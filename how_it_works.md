
---

## How the System Works

---

## Overview

This application is a **FastAPI-based PowerPoint generator** that:

1. Accepts raw user text
2. Uses an LLM (OpenAI / AI Pipe / Anthropic / Gemini) to structure it into slides
3. Enforces formatting and slide constraints
4. Builds a styled `.pptx` file (optionally from a template)
5. Returns the file as a downloadable presentation

The system is structured into three main layers:

```
API Layer → AI Planning Layer → PPT Rendering Layer
```

---

# 1. API Layer (FastAPI)

The application is built using FastAPI and exposes three main routes:

| Route          | Method | Purpose                       |
| -------------- | ------ | ----------------------------- |
| `/`            | GET    | Serves frontend HTML          |
| `/favicon.ico` | GET    | Prevents browser 404 errors   |
| `/generate`    | POST   | Generates the PowerPoint file |

---

## `/generate` Endpoint

This is the core of the system.

### It Accepts:

* `text` → Main content (required)
* `guidance` → Style/tone direction (optional)
* `provider` → LLM provider (`openai`, `aipipe`, `anthropic`, `gemini`)
* `api_key` → Provider API key
* `model` → Optional model override
* `num_slides` → Desired number of slides (1–40)
* `reuse_images` → Whether to reuse images from template
* `template` → Optional `.pptx` or `.potx` template

---

# 2. Input Validation

Before calling the AI:

### Text Validation

* Ensures text is not empty
* Truncates to 60,000 characters max

### Slide Count

* Clamped between 1 and 40
* Defaults to minimum 10 slides if not specified

### Template Validation

* Must be `.pptx` or `.potx`
* Minimum size: 1KB
* Maximum size: 30MB

---

# 3. AI Planning Layer

The system converts raw text into structured slides using an LLM.

---

## Step 1: Prompt Construction

The function `_llm_instruction()` creates a structured prompt that:

* Forces JSON-only output
* Defines slide format
* Enforces:

  * Title ≤ 80 characters
  * 3–6 bullets per slide
  * Bullet ≤ 120 characters
* Enforces exact slide count if requested

### Expected Output Format:

```json
{
  "slides": [
    {
      "title": "Slide Title",
      "bullets": [
        "Point 1",
        "Point 2",
        "Point 3"
      ]
    }
  ]
}
```

---

## Step 2: Provider-Specific Calls

Depending on the selected provider:

### OpenAI

Uses:

```
client.responses.create()
```

### AI Pipe

Uses:

```
requests.post() to OpenRouter-compatible endpoint
```

### Anthropic

Uses:

```
client.messages.create()
```

### Gemini

Uses:

```
generate_content()
```

All responses are parsed into JSON using `_safe_json_parse()`.

---

## Retry Logic

If the AI call fails:

* Retries up to 2 times
* Uses incremental backoff delay

This increases reliability for transient API failures.

---

# 4. Slide Count Enforcement

After receiving the AI plan:

## If `num_slides` is specified:

* The system forces the exact number using:

  ```
  enforce_target_slides()
  ```

## If not specified:

* Ensures at least 10 slides using:

  ```
  ensure_min_slides()
  ```

### Enforcement Logic Includes:

* Splitting slides with too many bullets
* Merging continuation slides
* Padding with empty slides if needed
* Truncating excess slides

---

# 5. PPT Rendering Layer

Slides are generated using the `python-pptx` library.

Main function:

```
build_presentation_from_plan()
```

---

## Template Handling

If a template is uploaded:

* The presentation loads template masters/layouts
* Existing slides are safely removed
* Template styling is preserved
* Images can optionally be reused

---

## Safe Slide Deletion

Slides are removed using:

```
_clear_all_slides_safely()
```

This drops slide relationships before deletion to prevent:

> "PowerPoint found unreadable content"

---

# 6. Slide Creation Process

For each slide in the AI plan:

1. Select Title + Content layout
2. Detect title placeholder
3. Detect body placeholder
4. Insert images (if enabled)
5. Insert title text
6. Insert bullet text

---

# 7. Smart Image Handling

If `reuse_images = True`, images from the template are copied.

The system ensures images never cover text.

### How?

#### Step 1: Collect Text Zones

`_collect_text_zones()` finds:

* Title placeholders
* Body placeholders
* Text boxes
* Subtitles

#### Step 2: Detect Overlap

`_overlaps_any_text()` calculates intersection area.

If overlap > 10%, image is repositioned.

#### Step 3: Choose Safe Zone

`_choose_safe_zone()` tries:

1. Right of body
2. Below body
3. Left side
4. Fallback sidebar

#### Step 4: Resize to Fit

`_fit_into_box()` scales image while preserving aspect ratio.

Images are inserted **before text**, ensuring text stays on top.

---

# 8. File Output

Presentation is saved into memory:

```python
out = io.BytesIO()
prs.save(out)
```

Returned using:

```
StreamingResponse()
```

With headers:

* `Content-Disposition: attachment`
* `Content-Type: application/vnd.openxmlformats-officedocument.presentationml.presentation`
* `Cache-Control: no-store`

The browser downloads:

```
generated.pptx
```

---

# System Flow Diagram

```
User Input
   ↓
/generate Endpoint
   ↓
Validate Input
   ↓
Build AI Slide Plan
   ↓
Enforce Slide Count
   ↓
Generate PPTX
   ↓
Stream Download
```

---

# Architectural Summary

## Layer 1 – API Layer

Handles:

* HTTP requests
* Validation
* Error handling

## Layer 2 – AI Planning Layer

Handles:

* Prompt engineering
* Provider abstraction
* JSON enforcement
* Retry logic

## Layer 3 – Rendering Layer

Handles:

* Template styling
* Slide layout selection
* Image safety
* Text placement
* File streaming

---

# Strengths of This Architecture

* Multi-LLM support
* Retry resilience
* Strict JSON enforcement
* Slide count control
* Template styling preservation
* Smart image placement
* Office-safe slide deletion
* In-memory file streaming
* Clean separation of concerns

---

# Extending the System

Possible improvements:

* Add caching layer
* Add async provider calls
* Add background tasks
* Add authentication
* Add rate limiting
* Add usage tracking
* Add SaaS billing layer
* Add persistent template library

---

# Conclusion

This application converts unstructured text into a structured, styled PowerPoint presentation by:

1. Using an LLM as a slide planner
2. Enforcing structural constraints
3. Rendering slides safely into a template-aware PPTX
4. Delivering a production-ready presentation file

It cleanly separates AI planning from document rendering, making it scalable and maintainable.

---
