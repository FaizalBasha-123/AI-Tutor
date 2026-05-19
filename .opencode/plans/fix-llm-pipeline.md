# Plan: Fix LLM Pipeline, Provider Integration & Structured Output Gaps

## Overview

Three categories of fixes, ordered by impact on production reliability:

---

## Phase 1 — Critical: Structured Output Enforcement

### Problem
Every provider request today sends unstructured prompts (`messages` only). No `response_format`, no `tools`, no `tool_choice`. The pipeline relies on fragile JSON parsing fallbacks (~900 lines in `response_parser.rs`). LLMs frequently return malformed JSON, especially Groq's smaller models.

### 1a. Add `ResponseFormat` type + update all provider request structs

**New file** (or add to `providers/src/lib.rs` or `domain/src/provider.rs`):
```
providers/src/request_params.rs
```

Define a shared `ResponseFormat` enum and a `GenerationParams` struct that all providers can accept:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum ResponseFormat {
    Text,
    JsonObject,
    JsonSchema { schema: serde_json::Value },
}

#[derive(Debug, Clone, Default)]
pub struct GenerationParams {
    pub response_format: Option<ResponseFormat>,
    pub tools: Option<Vec<ToolDefinition>>,
    pub tool_choice: Option<ToolChoice>,
    pub max_tokens: Option<u32>,
    pub temperature: Option<f32>,
}
```

The `ToolDefinition` and `ToolChoice` types can mirror OpenAI's existing shapes.

### 1b. Update `LlmProvider` trait (`providers/src/traits.rs`)

- Add `GenerationParams` parameter to all `generate_*` methods (as `&GenerationParams` or `Option<&GenerationParams>`)
- Add a default `None` so existing callers don't break
- The `generate_text` convenience method can use `&GenerationParams::default()`

The signature change:
```rust
async fn generate_text(
    &self,
    system_prompt: &str,
    user_prompt: &str,
    params: &GenerationParams,
) -> Result<String>;
```

With a backwards-compat default:
```rust
async fn generate_text(&self, system: &str, user: &str) -> Result<String> {
    self.generate_text_with_params(system, user, &GenerationParams::default()).await
}
```

### 1c. Update `OpenAiCompatibleProvider` (`providers/src/openai.rs`)

**Changes:**
1. Add `response_format`, `tools`, `tool_choice` fields to `ChatCompletionRequest`:
   ```rust
   #[serde(skip_serializing_if = "Option::is_none")]
   response_format: Option<ResponseFormat>,
   #[serde(skip_serializing_if = "Option::is_none")]
   tools: Option<Vec<ToolDefinition>>,
   #[serde(skip_serializing_if = "Option::is_none")]
   tool_choice: Option<ToolChoice>,
   ```
2. Populate these fields from `GenerationParams` in all 3 request construction sites (lines 111, 485, 539)
3. Ensure serialization matches OpenAI API spec (`response_format.type`, `response_format.json_schema.schema`, etc.)

### 1d. Update `AnthropicProvider` (`providers/src/anthropic.rs`)

**Changes:**
1. Add `thinking`, `tools`, `tool_choice` fields to `AnthropicRequest`:
   ```rust
   #[serde(skip_serializing_if = "Option::is_none")]
   thinking: Option<AnthropicThinkingConfig>,
   #[serde(skip_serializing_if = "Option::is_none")]
   tools: Option<Vec<AnthropicTool>>,
   #[serde(skip_serializing_if = "Option::is_none")]
   tool_choice: Option<AnthropicToolChoice>,
   ```
2. Populate from `GenerationParams` in all 3 request construction sites (lines 190, 265, 414)
3. Map `ResponseFormat::JsonObject` → don't set `thinking` (they're incompatible per Anthropic API)

### 1e. Update `GoogleProvider` (`providers/src/google.rs`)

**Changes:**
1. Add fields to `GoogleGenerationConfig`:
   ```rust
   #[serde(skip_serializing_if = "Option::is_none", rename = "responseSchema")]
   response_schema: Option<serde_json::Value>,
   #[serde(skip_serializing_if = "Option::is_none", rename = "maxOutputTokens")]
   max_output_tokens: Option<u32>,
   ```
2. Map `ResponseFormat::JsonObject` → `response_mime_type: "application/json"`
3. Map `ResponseFormat::JsonSchema { schema }` → `response_mime_type: "application/json"` + `response_schema: schema`

### 1f. Update `LlmGenerationPipeline` (`orchestrator/src/generation.rs`)

**Changes:**
1. Create a helper `fn generation_params() -> GenerationParams` that returns `{ response_format: Some(JsonObject) }`
2. Update all `generate_with_retry_using` calls to pass `&params`
3. For the web search tool loop: pass `GenerationParams { tools: ..., tool_choice: ... }` instead of relying on prompt markers like `TOOL_CALL: web_search`

### 1g. Update `resilient.rs` (`providers/src/resilient.rs`)

- Pass `GenerationParams` through the resilience layer to inner provider calls
- No structural changes needed — just thread the parameter through

---

## Phase 2 — Medium: OpenRouter Headers + Groq Handling

### 2a. Fix OpenRouter HTTP headers (`providers/src/openrouter.rs`)

**Current**: OpenRouter wrapper only appends text to the system prompt. Never sets HTTP headers.

**Fix**: The `OpenRouterLlmProvider` wraps an inner `Box<dyn LlmProvider>`. But HTTP headers are set at the HTTP client level in each provider's implementation (openai.rs, anthropic.rs, etc.). The wrapper cannot inject headers without intercepting the HTTP call.

**Options:**
- **A (recommended)**: Change how the provider is built in `factory.rs`. When wrapping with OpenRouter strategy, pass the OpenRouter base URL + headers to the inner provider's HTTP client configuration.
- **B**: Change OpenRouter to be a standalone provider (not a wrapper) that makes its own HTTP calls with proper headers.

**Recommended approach (A)**: 
1. In `factory.rs` `build()` method (around line 350-400 where OpenRouter strategy is applied):
   - When strategy is `OpenRouter`, set `default_base_url` to `https://openrouter.ai/api/v1`
   - Add `HTTP-Referer` and `X-Title` to the provider's HTTP client default headers
2. The inner provider (OpenAI-compatible) will then send all requests to OpenRouter with proper headers

### 2b. Groq-specific rate-limit detection (`providers/src/resilient.rs`)

**Current**: `is_rate_limited()` checks for `"429"` + `"Too Many"` / `"rate"` / `"limit"`. This catches most Groq rate limits since Groq uses standard HTTP 429.

**Fix**: 
1. Add Groq-specific error code detection to `is_rate_limited()`:
   ```rust
   // Groq sometimes returns 429 with "Request too large" or quota errors
   // OpenAI-compatible format, so the existing check should work
   // But add explicit Groq error body patterns:
   msg.contains("rate_limit_exceeded")
   ```
2. Add `is_groq_rate_limit()` helper that checks for Groq-specific response body patterns

**Note**: OpenRouter uses standard OpenAI-compatible error formats too, so existing detection should work for both.

---

## Phase 3 — Low: Fix Remaining Gaps

### 3a. Replace `panic!()` with proper error handling

**23 `panic!()` calls** — ALL in test code (see deep-dive analysis). They use `panic!()` as `unwrap()` on `Option`/`Result` in test assertions. These are not production code paths and won't crash the server.

**Fix**: Replace each with `expect()` or `?` propagation. No behavior change, just better test hygiene:
- `generation.rs` lines 3124, 3202, 3315, 3353, 3436, 3527, 3566, 3612, 3663
- `pipeline.rs` lines 1478, 1509, 1554, 1594, 1596, 1665, 1667, 1700, 1702, 1740, 1742

### 3b. Planner and Placement completion (`orchestrator/src/planner.rs`, `placement.rs`)

**Current state**: `planner.rs` has types/enums defined but execution path is minimal. `placement.rs` has phase enum but no placement test loop.

**Scope**: These are PBL (Project-Based Learning) mode features — not part of the core generation flow. Defer to a separate plan unless PBL is a priority.

### 3c. Prompt templating system

**Current**: All prompts are hardcoded Rust strings in `prompts.rs`.

**Scope**: Replacing with Tera/Handlebars would be a large refactor with no functional benefit (prompts still need recompilation to deploy). **Recommend deferring** until there's a need for non-developer prompt editing.

---

## Files Changed Summary

| File | Phase | Change |
|------|-------|--------|
| `providers/src/request_params.rs` (NEW) | 1 | `ResponseFormat`, `GenerationParams`, `ToolDefinition` types |
| `providers/src/traits.rs` | 1 | Add `&GenerationParams` parameter to all `generate_*` methods |
| `providers/src/openai.rs` | 1 | Add `response_format`/`tools`/`tool_choice` to request struct + populate |
| `providers/src/anthropic.rs` | 1 | Add `thinking`/`tools`/`tool_choice` to request struct + populate |
| `providers/src/google.rs` | 1 | Add `responseSchema`/`maxOutputTokens` to config + populate |
| `providers/src/resilient.rs` | 1 | Thread `GenerationParams` through to inner provider |
| `providers/src/factory.rs` | 1 | Ensure builder passes `GenerationParams` |
| `orchestrator/src/generation.rs` | 1 | Add `generation_params()` helper, pass to all LLM calls |
| `providers/src/openrouter.rs` | 2a | Inject HTTP headers via factory config |
| `providers/src/factory.rs` | 2a | Set OpenRouter base URL + headers when strategy is OpenRouter |
| `providers/src/resilient.rs` | 2b | Add `rate_limit_exceeded` to error detection |
| `orchestrator/src/generation.rs` | 3a | Replace 9 `panic!()` with `expect()` in tests |
| `orchestrator/src/pipeline.rs` | 3a | Replace 11 `panic!()` with `expect()` in tests |
| `orchestrator/src/planner.rs` | 3b | Deferred — PBL mode |
| `orchestrator/src/placement.rs` | 3b | Deferred — PBL mode |

---

## Implementation Order

```
Phase 1a → 1b → 1c + 1d + 1e (parallel) → 1f → 1g → 
  cargo check + cargo test →
Phase 2a → 2b → 
  cargo check + cargo test →
Phase 3a → 
  cargo check + cargo test + npm run build
```

---

## Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Adding `response_format` changes the LLM output behavior | Medium — models that currently produce near-valid JSON may produce different output when JSON mode is enforced | Test with a known working lesson prompt first; `response_parser.rs` fallback remains as safety net |
| `LlmProvider` trait change breaks external implementations | Low — all implementations are in-tree | Update all providers in the same commit |
| OpenRouter header injection via factory changes provider URL | Medium — if base URL is wrong, all traffic routes incorrectly | Verify OpenRouter base URL in staging before production |
| `panic!()` → `expect()` doesn't change behavior | None — purely cosmetic | No risk |

---

## Verification

```bash
# After Phase 1:
cargo check  # must pass
cargo test -p ai_tutor_routing --lib  # must pass 10/10

# After Phase 2:
cargo check
cargo test

# After Phase 3:
cargo check
cargo test  # all tests pass, no panics
npm run build  # frontend unaffected
```
