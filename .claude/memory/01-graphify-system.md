---
name: Graphify System for UC School
description: Graphify code metadata for UC School workspace
type: reference
---

# Graphify System - UC School

## What is Graphify

Graphify generates structured metadata about the UC School codebase so AI assistants can understand the project without manual file scanning.

## Output Files (in `.graphify-context/`)

| File | Purpose |
|------|---------|
| `project-context.txt` | File counts, stats, key root files, top projects |
| `project-tree.txt` | Clean directory structure (no node_modules/build artifacts) |
| `markdown-index.txt` | Categorized list of all documentation files |

## Accurate Project Structure

- **AI-Tutor-Backend/** — Rust workspace (Axum HTTP, not Node.js/Express)
  - `crates/api/` — Axum HTTP server, routes, handlers, auth, billing, queue
  - `crates/domain/` — Core models: lesson, billing, auth, scene, runtime, routing
  - `crates/orchestrator/` — AI lesson generation pipeline, pedagogy router, planner
  - `crates/providers/` — LLM / Image / Video / TTS / ASR provider factories
  - `crates/runtime/` — Multi-agent lesson playback engine
  - `crates/storage/` — PostgreSQL repositories, file system abstraction
  - `crates/media/` — Media task processing, asset storage (local + R2)
  - `crates/common/` — Shared utilities
- **AI-Tutor-Frontend/** — Next.js 16 + React 19 monorepo (pnpm workspace)
  - `apps/web/` — Main Next.js app (App Router)
  - `packages/types/` — Shared TS definitions
  - `packages/ui/` — Shared UI components
  - `packages/mathml2omml/` — MathML converter
  - `packages/pptxgenjs/` — PowerPoint generation
- **Schools24-backend/** — Go/Gin backend (school management, multi-tenant PostgreSQL)
- **Schools24-frontend/** — React + Capacitor frontend
- **schools24-landing/** — Landing page
- **OpenMAIC/** — Open-source AI classroom platform
- **graphbit/** — Knowledge graph / benchmark engine
- **zeroclaw/** — Hardware security / firmware project
- **client/** — Mobile client

## Stats

- TypeScript/TSX: ~33,142 files
- Rust: ~86 files
- Go: ~142 files
- Markdown (clean): ~569 files
- Total files: ~115,000 (including node_modules/build artifacts)

## AI Assistant Instructions

1. **ALWAYS read `.graphify-context/project-context.txt` first** when starting work on this codebase
2. **Reference `.graphify-context/project-tree.txt`** for directory structure before searching
3. **Check `.graphify-context/markdown-index.txt`** for relevant documentation before asking questions
4. **Remember**: AI-Tutor-Backend is Rust/Axum, NOT Node.js/Express
5. The project is at `/media/faizal-basha/Codespace/uc-school` (Linux), not `D:\uc-school` (Windows)

