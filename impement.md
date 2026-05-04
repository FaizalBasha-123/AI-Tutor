You are a senior product designer and frontend engineer.

Extend the existing AI Tutor pricing page by adding a **“Transparent Credit Usage” section below the pricing plans**.

This section must be clean, structured, and enterprise-grade (similar to Stripe / Notion documentation style). It should improve trust without overwhelming the user.

---

## 🎯 OBJECTIVE

Show users how credits are consumed across features in a **clear, readable matrix format**, without exposing technical complexity.

---

## 🧱 SECTION STRUCTURE

### Section Title:

“Transparent Credit Usage”

### Subtitle:

“Understand how credits are used across lessons, voice, and PDFs.”

---

## 📊 1. LESSON & MODES MATRIX

Create a table with:

Columns:

* Feature
* Basic
* Standard (highlight this column)
* Premium

Rows:

* Lesson Generation → 2 | 4 | 8 credits
* ⚡ Revision Mode → 1.2 | 2.4 | 4.8 credits
* 🧠 Explain Mode → 3.2 | 6.4 | 12.8 credits
* 📝 Exam Mode → — | 5.2 | 10.4 credits
* 🎯 Placement Mode → — | ~8 | ~16 credits

---

## 🔊 2. VOICE USAGE TABLE

Title:
“Voice Usage”

Columns:

* Feature
* Basic
* Standard
* Premium

Rows:

* TTS (per minute) → 0.4 | 0.8 | 1.5 credits
* ASR (per minute) → 0.5 | 1 | 1.5 credits

---

## 📚 3. PDF USAGE TABLE

Title:
“PDF Processing”

Columns:

* Plan
* Cost per page

Rows:

* Starter → 0.20 credits / page
* Pro → 0.15 credits / page
* Power → 0.12 credits / page

Add note below:
“Example: A 50-page PDF typically uses 8–11 credits depending on your plan.”

---

## 🎨 4. OPTIONAL MEDIA ROW

Add small section:

* Image generation → 3–8 credits

---

## 🧠 5. FOOTNOTE (IMPORTANT)

Add a subtle text block below all tables:

“Credit usage varies slightly based on content complexity and output size. Values shown are typical estimates.”

---

## 🎨 DESIGN REQUIREMENTS

* Use clean card-style containers for each table
* Rounded corners (large radius)
* Soft shadow
* Light gray background section
* Tables should be:

  * Well spaced
  * Easy to scan
  * Mobile responsive (stack rows or horizontal scroll)

---

## ✨ UX ENHANCEMENTS

* Highlight the “Standard” column (recommended usage)
* Do not use icons or subtle emojis for rows:

  * ⚡ 🧠 📝 🎯 🎧 📄
* Add hover effect on rows
* Keep text minimal and readable

---

## ⚠️ DO NOT INCLUDE

* No token-level details
* No model names (GPT, Claude, etc.)
* No backend explanations
* No formulas

---

## 🎯 FINAL GOAL

This section should:

* Increase trust through transparency
* Help users estimate usage quickly
* Make pricing feel predictable
* Encourage upgrading to Pro plan

---

## 📦 OUTPUT EXPECTATION

* Clean React + Tailwind implementation
* Reusable table components
* Responsive design
* Proper spacing and typography
* Production-level UI quality (not MVP)

---