---
name: course-summary
name_en: Course Summary Study Guide
description: Turn an enrolled online course (LinkedIn Learning, Udemy, Coursera, edX, etc.) into a professional PDF study guide by extracting the transcripts and course materials and rewriting them into a textbook-quality handbook. Use when the user wants a study guide, course notes, or a course summary produced from an online course and does NOT want to watch the videos.
description_en: Turn an enrolled online course (LinkedIn Learning, Udemy, Coursera, edX, etc.) into a professional PDF study guide by extracting the transcripts and course materials and rewriting them into a textbook-quality handbook. Use when the user wants a study guide, course notes, or a course summary produced from an online course and does NOT want to watch the videos.
argument-hint: Confirm the course is open in the browser (or give its URL); optionally set the output PDF path
argument-hint-en: Confirm the course is open in the browser (or give its URL); optionally set the output PDF path
user-invocable: true
version: 1.0.0
---

# Course Summary Study Guide

Extract a course's transcripts + materials and rewrite them into one polished PDF study guide the user can learn from without watching the videos. The transcript is **source material**, not the deliverable — never ship a reformatted transcript.

This is the reading complement to the `watchdog` skill (which plays videos to completion for certification). Use this skill when the goal is *learning by reading*, not *completing a course for credit*.

## Workflow

Copy this checklist and track progress:

```
- [ ] 1. Locate the course and map its full structure
- [ ] 2. Extract every transcript + useful material (no video watching)
- [ ] 3. Organize the content into a learning structure BEFORE writing
- [ ] 4. Write the study guide to the template in template.md
- [ ] 5. Render to a professional PDF (use the `pdf` skill)
- [ ] 6. Run the accuracy + quality checks, then deliver the file
```

## Phase 1 — Locate and map

1. Find the course tab. Use the browser tools (`mcp__builtin_browser`): `tabs_context` to list open tabs; if the course isn't open, `navigate` to the URL the user provides. If you cannot reach the course (not logged in, no URL), stop and ask — never pretend to have accessed it.
2. Read the curriculum outline (sections → clips, with titles and durations) via `read_page`. This is the spine of the guide; preserve the instructor's ordering because later concepts usually depend on earlier ones.

## Phase 2 — Extract the content (transcripts are the primary source)

Do **not** play or wait on videos. For each clip:

1. Open/load the clip in the player.
2. Reveal its transcript. On LinkedIn Learning the transcript lives in a tab in the panel beside/below the player (labeled "Transcript", sometimes behind a "..." / captions menu next to "Overview"). On other platforms find the equivalent captions/transcript control.
3. Capture the transcript text. Note: `get_page_text` does **not** return the transcript panel (it prioritises article content) — read the on-page Transcript tab's DOM instead (see the tactics below). Repeat per clip.
4. Also pull anything genuinely useful from the "Exercises & Downloads" / materials / description tabs (slide decks, exercise files, written descriptions).
5. Store the raw extraction to a working file (e.g. `transcripts.md`) grouped by section and clip, so you have a traceable source for the accuracy pass.

The platform UI changes; if a locator above doesn't match, discover the transcript control on the page and adapt — then, if this skill was wrong, patch the card with what you found (as `watchdog` does). Only the transcript drawer differs per platform; the rest of the workflow is platform-agnostic.

### Proven LinkedIn Learning tactics (hard-won — use these)

- **Do NOT trust the API transcript.** `GET /learning-api/detailedCourses?…&addTranscriptToChapterVideos=true&courseSlug=<slug>&q=slugs` returns each video's `transcript.lines`, but it is a **~60-second preview** (≈ first 27 lines), not the full text. Use it only to enumerate sections/clips/slugs/durations. The **full** transcript lives only in the on-page Transcript tab DOM.
- **The API is CSRF-guarded.** A bare `fetch()` returns `CSRF check failed`. Add header `csrf-token` = the `JSESSIONID` cookie value with quotes stripped (plus `x-restli-protocol-version: 2.0.0`). Still only a preview — enumerate with it, extract from the DOM.
- **Reliable per-clip loop** (the transcript renders async after the tab is clicked, so click and read must be **separate** JS calls — reading in the same synchronous call returns empty):
  1. `navigate` tool to the clip URL (this also waits for load and brings the tab to the foreground).
  2. JS: click the Transcript tab — `[...document.querySelectorAll('button')].find(e=>/^\s*Transcript\s*$/i.test(e.textContent)).click()`.
  3. JS: extract the largest `[class*=transcript]` node's `innerText`, collapse whitespace.
- **Best per-clip loop (single call, verified):** the `javascript_tool` **awaits a returned Promise**, so you can merge click + wait + read into ONE call instead of two. The Transcript tab/panel ids carry an ember prefix that changes on every page load (e.g. `hue-tabs-ember164-tab-TRANSCRIPT`), so always select by **suffix**:
  ```js
  new Promise(res=>{
    const b=document.querySelector('[id$="-tab-TRANSCRIPT"]'); if(b)b.click();
    setTimeout(()=>{
      const p=document.querySelector('[id$="-panel-TRANSCRIPT"]');
      let t=p?p.innerText:'';
      t=t.replace(/Selecting transcript[^\n]*\n?/,'').split('Enable interactive transcripts')[0];
      res(JSON.stringify({url:location.href,len:t.length,text:t.trim()}));
    },600);   // 600–700ms; a plain synchronous click+read returns empty because render is async
  })
  ```
  Enumerate clip slugs first: `[...document.querySelectorAll('a[href*="/learning/<course-slug>/"]')]`, and skip `.../quiz/...` links (chapter quizzes have no transcript).
- **Cut ~1 call per clip:** merge step 3 of clip *N* with the navigation to clip *N+1* by returning the transcript AND setting `location.href='<next clip url>'` in the same JS call; the tool round-trip gives the next page time to load. Make the following click robust (return `clicked:!!btn` and retry if `false`).
- **Throttled background tabs:** JS on a non-foreground tab can time out ("V2 command timeout"). Recover by calling the `navigate` tool on that tab (activates it), then retry.
- **Extension dropouts:** `builtin_browser` may disconnect mid-run and reconnect with **new tab IDs**. If a call errors with "Upstream server not found" or "No tab with given id", re-run `tabs_context`, find the course tab by title/URL, and resume — extraction is idempotent per clip so nothing is lost.
- **Garbled speaker names:** transcripts are auto-generated and mangle names (e.g. "Janani Ravi" → "Jenna V. Rubbi"). Normalise to the instructor name shown on the page.

## Phase 3 — Organize before writing

Before drafting, identify the main subject, learning objectives, core concepts, terminology, key principles, processes/workflows, frameworks/models, examples, practical techniques, best practices, common mistakes, important distinctions, real-world applications, and how concepts relate. Reorder so the material teaches cleanly while respecting prerequisite ordering (if B depends on A, A comes first).

## Phase 4 — Write the study guide

Build the full document to the exact structure, section list, writing style, and anti-cliché rules in **[template.md](template.md)** (cover → TOC → overview → roadmap → core concepts → lesson notes → difficult concepts → workflows → frameworks → glossary → application → mistakes → recaps → quizzes → final test → answer key → cheat sheet → "teach me in 10 minutes"). Follow it closely; it is the spec, not a suggestion.

Teach, don't summarize: explain concepts in your own precise language, combine related transcript sentences into coherent paragraphs, and reserve bullets for genuinely parallel items.

## Phase 5 — Render the PDF

Invoke the `pdf` skill to convert the finished Markdown/HTML into a clean, professional, textbook-style PDF: cover page, consistent typography and heading hierarchy, page numbers, headers/footers, callout boxes for key ideas, tables where they aid comprehension, clearly separated quizzes/answer keys, and simple diagrams only where a visual genuinely helps. Prioritize readability over decoration; don't overcrowd pages.

### Proven PDF-render tactics (Windows, hard-won)

- The `pdf` skill's cloud route may fail with `CLOUD_AUTH_REJECTED`; the local `markdown_to_pdf.py` (ReportLab) still works. Run it directly rather than `generate_mdx_pdf.py`.
- **Set `PYTHONIOENCODING=utf-8`** when invoking it from Git Bash — otherwise a `print` of the `→` arrow crashes on the cp1252 console before conversion starts.
- **Do not use YAML front-matter** for the cover: the local renderer prints it as literal text on page 1. Put cover info in normal Markdown instead.
- **Avoid `$$…$$` LaTeX** — it renders as raw source. Write formulas as fenced code blocks with Unicode (`σ`, `Σ`, `∈`, `·`, `^(l)`); inline `code` spans render cleanly.
- Tables, callout blockquotes, numbered lists, and page numbers (`Page N / M`) all render well. Validate with `validate_pdf.py` and spot-check a few pages via `convert_pdf_to_images.py` + vision before delivering.

## Phase 6 — Quality and accuracy checks (run before delivering)

Accuracy (non-negotiable):
- Never invent facts, examples, stats, or claims and attribute them to the course.
- If a transcript was unavailable, say so and mark the gap rather than filling it.
- Preserve important technical detail and nuance; don't shorten away meaning.
- Clearly label any helpful context you add that is NOT from the course.

Instructional-design pass — ask and fix: Would someone who never watched this understand it? Are hard concepts actually explained (not just named)? Do quizzes test understanding/application rather than verbatim recall? Are answers correct? Is it easy to navigate? Any repetition or transcript-ese to cut?

Deliver with `qwenwork_file_present_files` (the PDF, and the raw `transcripts.md` only if the user wants the source). Keep the closing note short.
