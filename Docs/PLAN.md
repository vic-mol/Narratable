# PDF → EPUB for ElevenReader: Project Plan

Nov 4, 2026 · @Victor

## Summary

Build a Python desktop app that converts born-digital PDFs into narration-ready EPUB 3 files. It should run fully offline on a CPU-only Windows PC. Rule-based cleanup handles the bulk of the text, and a small local LLM is used only for the few things rules can't do (table summaries).

The project targets **ElevenReader's free plan only**. Every book is trimmed and split so its listening time fits the free hours, and any audio file the app makes is generated offline at no cost.

**Decisions so far**

| Question | Your answer | What it means for the design |
| --- | --- | --- |
| Interface | Desktop app | Python core with a thin GUI on top; ship a Windows .exe |
| Stack | Python | Fits: the best PDF and layout libraries are Python-first |
| PDFs | Born-digital technical books (sample: *AI Engineering*, 535 pages) | Read the embedded text layer; OCR is a later fallback, not the core |
| Language | English | One language to tune for: hyphenation, quotes, and abbreviation handling for English |
| Hardware | i5-9500, 32 GB RAM, Intel UHD 630, no NVIDIA GPU | CPU-only, so GPU-heavy layout models are slow; small local LLMs are usable for short jobs only |
| Cloud | Offline only | No OCR or LLM APIs; every model runs locally |
| ElevenReader plan | Free tier only | Budget every book against 10 free hours; no offline downloads, no audio export, no smart import (see *Designing for the free tier*) |
| Review | Fully automatic | The pipeline must check its own output and write a report instead of asking you |
| Footnotes | Read inline | Move each footnote's text to right after the sentence that cites it, marked as an aside |
| Tables | Summarize | Turn each table into a few spoken sentences (local LLM), and keep the real table in a part narration skips |
| Listening | ElevenReader audio → MP3 on phone, plus the EPUB for reading | ElevenReader can't export audio on any plan, so MP3s come from an offline TTS stage (Kokoro); ElevenReader is for streaming in the app |

## Repository layout and sample files

- `Docs/PLAN.md`: this plan.
- `References/`: **sample PDFs to convert**, starting with `References/AI Engineering.pdf` (book #1 in the test set). Coding agents and contributors should open these to check real layout before writing or changing extraction rules: fonts, running footers, footnotes, tables, sidebars and code. Render a page to an image, or dump its spans with PyMuPDF. New sample PDFs will be added here over time.
    - The folder is **git-ignored** (`/References/` in `.gitignore`) because the books are copyrighted. Read from it, but never commit it. Never paste its text into committed tests, fixtures, docs or commit messages.
    - Snapshots or EPUBs generated from these files go in a git-ignored `testdata/private/` folder. Add that folder to `.gitignore` in Milestone 1.
- `testdata/` (to be created): openly licensed or self-generated test PDFs and their expected outputs, safe to commit.

## What ElevenReader actually does with imports

**The ElevenReader app cannot export audio, so the “ElevenReader → MP3” step won't work as planned.** ElevenLabs' help center says the app is “only for personal listening and reading” and that audio can be created and exported only on its web platform ([help article](https://help.elevenlabs.io/hc/en-us/articles/26165470379537-Can-I-download-audio-from-ElevenReader)). The plan below adds a fully offline MP3/M4B option to replace it.

| Topic | What's documented | Source |
| --- | --- | --- |
| Import formats | EPUB, PDF and TXT files, plus pasted text, camera scans and URLs | [How do I add content](https://help.elevenlabs.io/hc/en-us/articles/26197616307985-How-do-I-add-content-to-ElevenReader) |
| Size limit | “Uploads of files up to 500 pages”; longer documents must be split | [Max file length](https://help.elevenlabs.io/hc/en-us/articles/44397769907217-What-is-the-maximum-file-length-supported-in-ElevenReader) |
| Free plan (**our target**) | 10 hours of text-to-speech for your uploads, which ElevenLabs puts at “nearly 400 pages”. Hours are spent only the first time text is converted; replays are free. Hours are counted at 1.0× speed. No offline downloads | [Pricing](https://elevenreader.io/pricing), [How hours work](https://help.elevenlabs.io/hc/en-us/articles/35971782968465-How-do-ElevenReader-hours-work), [v4 announcement](https://elevenreader.io/blog/introducing-elevenreader-v4) |
| Ultra plan (not used here) | $11/month: up to 24 h of generated audio per day, 150 h/month of in-app offline downloads, and “smart file imports” that skip headers, footers and filler | [Pricing](https://elevenreader.io/pricing) |
| Voice engine | Eleven v4 since Oct 2026, with better handling of abbreviations and pauses; in-app custom pronunciations | [v4 announcement](https://elevenreader.io/blog/introducing-elevenreader-v4) |
| Audio export | Not in the app. ElevenLabs Studio (web, paid credits) imports EPUB, splits chapters on Heading 1, and creates audiobooks there | [Studio import formats](https://help.elevenlabs.io/hc/en-us/articles/25708839235345-Which-file-formats-can-I-import-with-Studio) |

**What this means for the design**

- **One file per heading level 1.** Each chapter gets its own XHTML file starting with an `<h1>`. That matches what ElevenLabs Studio uses to split chapters, and it's standard EPUB practice.
- **Splitting into volumes.** At 535 PDF pages, *AI Engineering* may go over the 500-page limit. ElevenLabs doesn't say how it counts pages in an EPUB, so the app needs a split option (by Part or by chapter count). The first test will check whether splitting is actually needed.
- **Your listening budget.** The free plan's hours shape several features; see the next section.

**Undocumented behavior (Milestone 0 tests it):** ElevenLabs publishes nothing on how ElevenReader treats EPUB footnote markup (`epub:type="footnote"`), tables, image alt text, `<pre>` code, figure captions, CSS-hidden text, or pronunciation lexicons. Rather than guess, the first milestone imports a small probe EPUB and records what gets read aloud.

## Designing for the free tier

We build and test against ElevenReader's **free plan only**. Everything the app produces must work without paying, and development must not burn through the free hours.

**What the free plan gives, and what it doesn't**

- 10 hours of generated audio for your imports. ElevenLabs is inconsistent about the reset: the pricing page says monthly, and the hours article says both “each week” and “per month”. The plan assumes the stricter **10 hours per month** until Milestone 0 confirms it.
- Hours are spent only when text is converted to audio the first time. Replaying is free, and hours are measured at 1.0× speed.
- There are no offline downloads, so listening in the app needs a connection. There's no audio export on any plan. And “smart file imports” (skipping headers, footers and filler) are Ultra-only, so on the free plan our cleanup is the only cleanup.

**What that means for *AI Engineering***

The ten chapters hold about 162,500 words of raw PDF text. At an assumed narration pace of ~155 words per minute, that's about 17.5 hours before cleanup. After removing code, URLs and the index and summarizing tables, I'd expect roughly 15 hours. That's about two months of free hours, or about two weeks if the reset turns out to be weekly. With a 9-hour cap per volume, the split would be:

| Volume | Contents | Estimated hours (raw text) |
| --- | --- | --- |
| 1 | Preface, Chapters 1–4 | \~8.0 |
| 2 | Chapters 5–8 | \~7.0 |
| 3 | Chapters 9–10, Epilogue | \~3.0 |

These numbers come from raw word counts and an assumed pace; Milestone 0 measures the real rate.

**Features this adds to the MVP**

1. **Listening-time estimate.** The report shows words and estimated hours per chapter, per volume and in total. It uses a words-per-minute rate calibrated in Milestone 0.
2. **Budget-aware volume split.** A setting, `--hours-per-volume`, defaults to 9, which leaves about an hour a month for testing and short articles. The split happens at chapter boundaries and writes one EPUB per volume, titled e.g. “AI Engineering (Vol. 1 of 3)”, each with full metadata and a cover. This also keeps each volume under the 500-page upload limit.
3. **Lean narration.** Anything not worth listening time stays out of the spoken text:
    - the cover, copyright page and printed TOC (the EPUB's own navigation replaces it)
    - the index and colophon
    - raw URLs
    - code, which becomes a one-line spoken cue
    Optional switches control parenthetical citations (“Devlin et al., 2018”) and footnotes; reading footnotes inline costs roughly 0.5–1 hour on this book (my estimate).
4. **Import once.** Replaying converted audio is free, but an edited EPUB imported again is almost certainly new text. So the app should get a book right before you import it. The lint must pass, and only then does the report say “ready to import”.

**Testing without spending hours**

- The Milestone 0 probe EPUB is tiny, a few minutes of audio.
- For day-to-day listening checks while developing, use a local voice (Kokoro, or Windows' built-in voice) on excerpts. Only release candidates go into ElevenReader.
- ElevenReader import tests use a ~10-minute excerpt EPUB with a chapter start, a page-break join, a footnote, a table summary and a sidebar.
- Keep a short hours log in the repo (`Docs/hours-log.md`): date, what you imported, and the hours used.

**If you're on another plan**

The app works the same for people on other plans; only the budget settings change. Neither of these is needed for this project:

- **Ultra** ($11/month) effectively removes the hour budget, adds offline downloads in the app, and does its own smart import. Set `--hours-per-volume 0` to turn splitting off.
- **Hour packs** can be bought without a subscription ([How hours work](https://help.elevenlabs.io/hc/en-us/articles/35971782968465-How-do-ElevenReader-hours-work)).
- **ElevenLabs Studio** (web, paid credits) can turn the EPUB into an exportable audiobook. The one-`<h1>`-per-chapter layout already suits it.

## What the sample PDF tells us

*AI Engineering* (Chip Huyen, O'Reilly, 535 pages) is a clean, born-digital, typeset book. Every page has a real text layer, so this book needs no OCR. The hard parts are structure and noise, not reading the characters. I inspected it with PyMuPDF.

| Feature | What I found | What the pipeline must do |
| --- | --- | --- |
| Bookmarks | 182 entries over 3 levels (Part → Chapter → section), with page numbers | Use the bookmarks as the chapter and TOC backbone instead of guessing headings |
| Page labels | Roman numerals for front matter, Arabic for the body | Strip page numbers; keep the labels only for debugging |
| Running footers | e.g. “4 \| Chapter 1: Introduction to…” and “The Rise of AI Engineering \| 7” at a fixed height | Remove text in the top and bottom bands when it repeats across pages |
| Line-end hyphenation | 1,615 breaks use the Unicode hyphen U+2010 (“struc‐ / ture”) and about 150 use an ASCII hyphen (real compounds like “self-”) | Join U+2010 breaks with no hyphen and keep ASCII ones. This typesetter makes the two cases tell themselves apart. |
| Footnotes | About 250 footnotes. Markers are small superscript digits glued to words (“blue.”3”), and the bodies sit at 8 pt below a rule | Detect markers by size and position, pair each with its body, and move the body inline |
| Sidebars | Tip and note boxes with an animal icon, set at 9.6 pt vs 10.5 pt body text | Detect by font size and the icon; read them as a short aside (“Note: …”) |
| Tables | About 46 pages have numbered tables (“Table 1-1. …”), often with code-font cells | Extract the cells, then summarize them for narration |
| Figures | About 153 pages with figure captions and 224 embedded images | Keep the image and caption in the EPUB, and narrate only the caption |
| Code | 69 pages use the monospace font (Ubuntu Mono) | Keep the code visible in the EPUB, and replace it in narration with a short spoken placeholder |
| Links | About 3,500 links: cross-references (“Chapter 2”, “Table 1-1”), citations, URLs | Keep the link text and drop raw URLs from narration |
| Lists and definitions | Numbered lists and italic term + indented definition blocks | Rebuild as `<ol>` and `<dl>` so paragraphs don't merge |

Since the bookmarks, fonts and positions are so regular, most of this is solvable with **deterministic rules on the PDF's own text layer**, which is fast and exact. That shapes the stack below.

## Options compared

On a CPU-only i5-9500, read the PDF's own text layer with PyMuPDF and write the rules yourself. Keep Docling as a fallback for messy layouts, and use a small local LLM only for table summaries. The ML-heavy converters produce Markdown aimed at RAG, not narration, and they're 30–1,000× slower without a GPU.

**PDF extraction and layout**

| Tool | What it gives you | Speed (rough, per page) | Licence | Role here |
| --- | --- | --- | --- | --- |
| [PyMuPDF 2.x](https://pypi.org/project/pymupdf/) (now includes the former pymupdf4llm) | Every character with font, size, bold/italic, position and colour; bookmarks, links, images, vector lines; `find_tables()`; Markdown/JSON output; optional OCR | \~0.01 s ([pdfmux](https://pdfmux.com/blog/pdfmux-vs-pymupdf-vs-marker-vs-docling/)) | AGPL-3.0 or commercial | **Primary extractor** |
| [PyMuPDF Layout](https://openapps.pro/packages/pymupdf-layout) | A small graph neural net, run through ONNX on CPU, that labels headings, page headers, footers and tables from the PDF's own text and vectors, not from images | Fast, no published figure | AGPL-3.0 | Optional helper for the role-labelling step |
| [Docling](https://github.com/docling-project/docling) (IBM / LF AI) | Layout model, reading order, table structure, OCR; lossless JSON | 0.3–3 s; best overall in the [pdfmux benchmark](https://pdfmux.com/blog/pdfmux-vs-pymupdf-vs-marker-vs-docling/) (0.877) | MIT | **Fallback** for PDFs without bookmarks, multi-column pages, and scans |
| [Marker](https://github.com/datalab-to/marker) | Markdown/HTML/JSON; removes headers and footers by default; strong on math (LaTeX); optional LLM mode (cloud) | 5–10 s on CPU | Code Apache-2.0; model weights restricted (free for personal use) | Not recommended: slow on your CPU, and the good LLM mode is cloud-based |
| [MinerU](https://github.com/opendatalab/MinerU) 4.x | Strongest on complex layouts, tables and Chinese/Japanese/Korean; ONNX on CPU | Moderate; GPU recommended for best quality | Apache-2.0 with extra terms | Not needed for English books |

**OCR (scanned PDFs, a later milestone)**: [OCRmyPDF](https://ocrmypdf.readthedocs.io/) with Tesseract adds a text layer to a scanned PDF, so the rest of the pipeline stays the same. PyMuPDF's docs recommend RapidOCR (ONNX) as the more accurate engine. Docling's OCR pipeline also runs offline. Expect several seconds per page on your CPU.

**Local LLM cleanup**: [small 3–4B models](https://www.popularai.org/p/best-cpu-only-local-llm-2026) (e.g. Qwen3.5 4B, Phi-4-mini, Gemma 4 E4B) at Q4\_K\_M quantization run on CPU through llama.cpp or Ollama. My estimate for your i5-9500 and 32 GB DDR4 is roughly 6–12 tokens/s output. That makes them good for about 50 table summaries per book (about 20 minutes total). Rewriting the whole book (\~350k tokens) would take 10+ hours and could quietly change facts. So the LLM handles tables only, never the main text.

**EPUB generation**

| Option | Notes | Verdict |
| --- | --- | --- |
| Write it yourself (Jinja2 templates + `zipfile`) | An EPUB is a zip of XHTML files, an OPF manifest and a nav file, about 150 lines of code. You control every `epub:type`, class and file split | **Recommended** |
| [EbookLib](https://github.com/aerkalov/ebooklib) | Convenient Python API, but AGPL, and its EPUB 3 nav output is fiddly | Fine for prototyping |
| Pandoc | Excellent HTML/Markdown → EPUB 3, but semantic markup goes through its own Markdown model | Good for a quick spike |

**Validation**: [EPUBCheck 5.4.0](https://github.com/w3c/epubcheck/releases) (W3C, Java) is the standard validator.

**Offline audio (replacing the ElevenReader MP3 step)**: [Kokoro-82M](https://pypi.org/project/audiblez) (Apache-2.0) is a small text-to-speech model that produces natural audio on CPU. audiblez reports about 60 characters/s on an M2 MacBook CPU. Your i5-9500 is probably slower, so I estimate 8–15 hours for this book. That's an overnight job, with voices a step below ElevenLabs. ElevenLabs Studio sounds better but needs paid credits and runs in the cloud, so it's out of scope here.

## Scope

The MVP turns *AI Engineering* into an EPUB that passes EPUBCheck. ElevenReader should read it with no page artifacts, footnotes read inline, and tables summarized, all offline and with no clicks after “Convert”. Each volume must fit within the free plan's hours.

**MVP**

- Born-digital PDFs with a text layer, single-column layout, English
- Chapters and TOC from the PDF's bookmarks, with fallback to font-size heading detection
- Remove running headers, footers and page numbers; join hyphenated words and split paragraphs across lines and pages
- Footnotes moved inline after the citing sentence; sidebars read as “Note: …” asides
- Tables summarized by a local LLM, with a rule-based fallback sentence if the LLM is off or fails
- Figures kept with their caption; code blocks kept on screen and replaced by a one-line spoken cue
- Unicode cleanup, `xml:lang="en"`, title, author and cover taken from the PDF
- Free-tier features: listening-time estimate, lean narration (no front or back matter), and splitting into volumes by hours
- EPUBCheck plus a narration lint report; desktop app with drag-and-drop, options, progress and “open output”

**Later**

- Offline audio export: Kokoro → one MP3 per chapter + an M4B with chapter marks
- Two-column pages, PDFs without bookmarks and Docling fallback mode
- Scanned PDFs (OCR), math (spoken form + image), other languages
- Plan profiles for people on Ultra or with hour packs (no splitting, fuller narration)
- Custom pronunciation list (“LLM” → “L L M”, “GPT-4o” → “G P T four oh”) applied to the narration text
- A Mac build and sharing it publicly. Licensing matters here: PyMuPDF is AGPL, so a public release must be open source under AGPL, or you swap the extractor

## Architecture and pipeline

The core is a Python library with a CLI. The desktop app is a thin shell that calls the same functions. Every stage reads and writes one intermediate **document model** (JSON). That way each stage can be tested alone, and a bad output can be traced back to the stage that caused it.

&#91;embedded content: conversion pipeline · 4 input routes, 3 outputs\]

The MVP uses only the native route. The dashed routes and audio come in later milestones, and all of them reuse the same downstream stages.

**Stages**

1. **Probe.** Open the PDF and record page count, bookmarks, page labels, fonts, and how many pages have text vs only images. Pick a route: *native* (text layer and bookmarks), *native-no-bookmarks* (detect headings from fonts), *layout-fallback* (Docling), or *scanned* (OCR first).
2. **Extract.** PyMuPDF `get_text("rawdict")` gives every span with font, size, flags, colour and bounding box. Also collect images, link targets, vector lines (footnote rules, table borders), and `find_tables()` candidates.
3. **Classify.** Give each line a role using font statistics and position:
   - `body`: the most common size
   - `heading`: matched to a bookmark title
   - `running_header`/`running_footer`: top or bottom band, repeated across pages
   - `page_number`
   - `footnote_body`: small text below a rule
   - `footnote_marker`: superscript digit in body text
   - `caption`: “Figure n-n.” or “Table n-n.”
   - `sidebar`: smaller text plus an icon
   - `code`: monospace font
   - `table_cell`, `list_item`, `def_term`
4. **Order and join.** Sort blocks into reading order (column-aware). Remove the running headers and footers. Join U+2010 line-break hyphens and keep real ones. Join lines into paragraphs, and join paragraphs that continue across a page break or are interrupted by a figure or footnote.
5. **Structure.** Build the tree Book → Part → Chapter → Section from the bookmarks. Attach figures, tables and code to their place in the flow.
6. **Narration transforms.** Move each footnote inline after its sentence. Summarize tables with the LLM. Turn captions into spoken lines, and code into one-line cues. Normalize Unicode (ligatures, odd spaces, glyphs TTS reads badly) and drop raw URLs from spoken text.
7. **Build EPUB.** Generate one XHTML file per chapter starting with `<h1>`, plus `nav.xhtml`, a legacy `toc.ncx`, an OPF with metadata, the cover, CSS and images. Then zip it, with `mimetype` stored first and uncompressed. If the book is over the hours budget, write one EPUB per volume.
8. **Validate and report.** Run EPUBCheck and the narration lint, then write `report.html` with warnings, stats, estimated listening hours per chapter and per volume, and a sample of before/after paragraphs.
9. **Audio (optional, later).** Feed the narration text per chapter to Kokoro, then use ffmpeg to make MP3 per chapter plus an M4B with chapter marks.

**Document model (sketch)**

```json
{"meta": {"title": "AI Engineering", "authors": ["Chip Huyen"], "lang": "en", "cover": "cover.jpg"},
 "chapters": [{"id": "ch01", "title": "Introduction to Building AI Applications…", "src_pages": [25, 72],
   "blocks": [
     {"type": "heading", "level": 2, "text": "The Rise of AI Engineering"},
     {"type": "paragraph", "text": "… favorite color is __.", "footnotes": [{"n": 3, "text": "Autoregressive language models are sometimes referred to as causal language models."}]},
     {"type": "table", "label": "Table 1-1", "caption": "…", "rows": [["Input (context)", "Output (next token)"]], "spoken": "…"},
     {"type": "figure", "label": "Figure 1-2", "image": "img/f1-2.png", "caption": "…"},
     {"type": "code", "lang": "python", "text": "…", "spoken": "A code example follows."},
     {"type": "aside", "kind": "note", "blocks": ["…"]}]}]}
```

Each block also keeps its source page and bounding box. That gives the debug viewer and the lint report a way to point back to the exact spot in the PDF.

## Recommended stack

Python 3.12, PyMuPDF for extraction, hand-written EPUB templates, llama.cpp for table summaries, and PySide6 for the desktop app. Package it as a Windows .exe with PyInstaller.

| Layer | Choice | Why |
| --- | --- | --- |
| Language / env | Python 3.12, managed with `uv` | Fast installs, lockfile, same setup on Windows and Mac |
| Extraction | `pymupdf` 2.x | \~1,000× faster than ML parsers on CPU; exposes the font, size and position data the rules rely on |
| Fallback layout | `docling` (optional extra) | MIT licence, CPU-first, good reading order and tables; only for PDFs the rules can't handle |
| Text helpers | `regex`, `wordfreq` (dehyphenation tie-breaks), `pysbd` (sentence boundaries for inserting footnotes), `ftfy` (Unicode repair) | Small, pure-Python, well-tested |
| Local LLM | `llama-cpp-python` with a 3–4B instruct GGUF at Q4\_K\_M; Ollama works too | Runs in-process with no server; the model is a swappable setting |
| EPUB | `jinja2` templates + `zipfile` | Full control of markup; no AGPL EPUB library needed |
| Validation | EPUBCheck 5.x (Java 11+), called as a subprocess | The official validator; skipped with a warning if Java is missing |
| Desktop UI | `PySide6` (Qt, LGPL) | Native look on Windows and Mac, background worker threads, embedded web view for the HTML report |
| CLI | `typer` | The same functions as the GUI; makes scripting and testing easy |
| Tests | `pytest` + snapshot tests on the document model and narration text | Catches regressions as the rules grow |
| Packaging | PyInstaller one-folder build (Nuitka later if startup is slow) | Well-trodden for PySide6 on Windows; the LLM model file downloads on first use instead of being bundled |
| Audio (later) | `kokoro` + `ffmpeg` | Apache-licensed, offline, CPU-capable |

The exact package versions are unverified. Pin whatever is current when you start, and record them in the lockfile.

## Hard cases

**Footnotes read inline (MVP).** A marker is a span much smaller than the body text, raised above the line, containing only digits, usually right after punctuation. A footnote body is small text below a horizontal rule that starts with the same number. Pair them by number within the page; if there's no match, check the next page. Insert the body right after the end of the sentence that holds the marker, not mid-sentence. Wrap it in spoken bookends so the listener knows where it starts and stops:

```html
<p>…It predicts what comes next in “My favorite color is __.”
  <span class="fn-inline">Footnote: Autoregressive language models are sometimes referred to as causal language models. End of footnote.</span>
  An autoregressive model can continually generate one token after another.</p>
```

Don't use `epub:type="footnote"` here. Some readers hide content marked that way, which is the opposite of what you want; Milestone 0 confirms this for ElevenReader. A setting can switch to *end of chapter* or *drop*. For footnotes that are only a citation or URL, read just the link text.

**Tables summarized (MVP).** Get cells from `find_tables()`. Because many of these tables are borderless code-font grids, fall back to column clustering on x-positions. Send the caption plus the cells as Markdown to the local LLM, with a strict prompt: 2–4 sentences, no new facts, mention the row count. Then check the summary: every number and proper noun it contains must appear in the table. If the check fails, use the rule-based fallback (“Table 1-1, Training samples for language modeling: two columns, six rows.”). In the EPUB, show the spoken summary as text and the real table as an image with empty alt text. Readers still see the full table, and TTS can't trip over cell soup. Milestone 0 confirms ElevenReader skips empty-alt images.

**Figures and code.** Narration reads “Figure 1-2: …caption…” and skips the image. Code blocks keep their exact text on screen, styled as `<pre>`. If Milestone 0 shows ElevenReader reads `<pre>`, render the code as an image as well, and narrate a cue such as “A Python code example follows.” Inline code words like `<BOS>` get a pronunciation rule (“B O S token”).

**Sidebars.** Detect the smaller font plus the animal icon, and output an `<aside>` that starts with a spoken label: “Note:”, “Tip:” or “Warning:”.

**Two-column layouts (later).** Make a histogram of block x-positions per page. A clear gap near the middle means two columns, so read the left column top to bottom, then the right. Full-width blocks such as titles and wide figures cut across both columns and reset the order. If that fails, route the page to Docling's reading-order model.

**Scanned PDFs (later).** Probe flags pages with images but under \~50 characters of text. Run OCRmyPDF (Tesseract) or Docling with RapidOCR to add a text layer, then reuse the normal pipeline. Font roles are weaker after OCR, so lean on position and repetition. Expect 3–10 s per page on your CPU, so about an hour for a 500-page book.

**Math (later).** Detect math fonts (e.g. CMMI/CMSY) or symbol-dense spans. Render the formula as an image for readers, and narrate a spoken form generated by the LLM, then checked against the symbols. *AI Engineering* has little math, so it's not an MVP blocker.

**Non-English (later).** The language code flows into `xml:lang` and the OPF. Dehyphenation and sentence splitting need per-language data (`pyphen`, `pysbd` support many languages). The LLM prompts need translating.

## Validation

With no manual review, the app has to grade its own output. Every run produces a pass/warn/fail verdict from four checks, and you only listen to the samples it points you to.

1. **EPUBCheck.** Zero errors is required, and warnings are listed in the report. This catches broken nav, missing manifest items, bad XHTML and missing metadata.
2. **Narration lint.** These are cheap regex and statistics checks on the spoken text. Each hit links back to its PDF page:
   - leftover line-break hyphens (`\w[‐­-]\s\w` in the middle of a paragraph)
   - digits glued to words or punctuation (`[a-z.”]\d{1,3}\b`), which signal missed footnote markers
   - running-title strings or bare page numbers that survived
   - paragraphs ending without terminal punctuation, followed by a lowercase start, which signal a missed join
   - paragraphs over \~400 words, which signal a missed split
   - URLs, characters outside an allowlist, private-use glyphs, ligature code points
   - footnote pairing rate: markers found vs bodies found vs bodies inserted
   - chapter count vs bookmark count; every chapter non-empty
3. **Ground-truth diff.** On the round-trip test books (next section), compare the output text to the original source text: character error rate, plus counts of missing and duplicated paragraphs. Target: under 0.5% character error and zero missing paragraphs.
4. **Listening checks.** You do these. The report picks five spots per book: a chapter start, a page break mid-paragraph, an inline footnote, a table summary and a sidebar. Import the EPUB into ElevenReader and jump to each one via the TOC. That's about 10 minutes per book instead of hours. On the free plan this should cost only the minutes you actually play, since hours are spent as text is first converted. Log what sounded wrong in a short checklist, and turn each finding into a new lint rule or a test.

**ElevenReader import test (every release, using the ~10-minute excerpt EPUB so it costs minutes, not hours):** check the import succeeds, the title, author and cover show, the TOC lists every chapter, and chapter jumps land on the right heading. Make sure the first page doesn't read out copyright or TOC boilerplate. Mark the cover, title page and TOC with `linear="no"` in the spine if ElevenReader respects it (Milestone 0).

## Test set

Use eight PDFs, including two you generate yourself so the right answer is known. Copyrighted PDFs live in `References/` and their outputs in `testdata/private/`, both git-ignored. Openly licensed PDFs and their expected-output snapshots go in `testdata/` and are committed.

| # | PDF | Edge cases covered | Where to get it |
| --- | --- | --- | --- |
| 1 | *AI Engineering* (your sample) | Bookmarks, running footers, U+2010 hyphenation, 250 footnotes, sidebars, code, 46 table pages, 150+ figures | `References/AI Engineering.pdf` |
| 2 | **Round-trip novel**: a public-domain novel printed to PDF with headers, footers, page numbers and justified hyphenation turned on | Paragraph joins across pages, dialogue quotes, chapter detection, with exact ground truth | A Standard Ebooks or Project Gutenberg EPUB → print to PDF from Calibre or a browser |
| 3 | **Round-trip with footnotes**: an annotated public-domain text with real footnotes, exported the same way | Footnote pairing and inline placement, with exact ground truth | Same as #2 (choose an edition with notes) |
| 4 | An open textbook | Tables, figures, sidebars, some math, lists; heavier layout | OpenStax (CC BY) |
| 5 | A two-column research paper | Two-column reading order, equations, references section | arXiv |
| 6 | A long government or policy report | Numbered headings, dense footnotes, large tables, and possibly no bookmarks | Any public agency report |
| 7 | A scanned public-domain book | OCR route, page skew, no bookmarks, OCR'd running headers | Internet Archive |
| 8 | A PDF without bookmarks and with a non-standard heading style | Heading inference from font size/weight | Any self-published or older PDF you own |

The MVP must pass #1–#3. #4 and #6 should come out usable with warnings. #5, #7 and #8 cover the later milestones.

## Milestones

Build order: learn ElevenReader's real behavior first, then get a plain listenable EPUB end to end, then add the hard parts one at a time. Leave the GUI until the core is trustworthy. Effort is a rough hobby-pace guess.

| # | Milestone | Deliverable | Done when | Effort |
| --- | --- | --- | --- | --- |
| 0 | ElevenReader probe | A hand-written \~10-page EPUB with one test per construct: `epub:type` footnote, `<aside>`, `<table>`, `<pre>`, empty-alt and non-empty-alt images, `figcaption`, CSS-hidden text, `linear="no"`, ligatures, smart quotes, “LLM”/“GPT-4o”. Plus a long dummy EPUB to probe the 500-page limit (check whether the upload is refused before any audio is generated). Also measure hours: import a text of known word count, note the hours deducted, and check whether the reset is weekly or monthly | A one-page table of what ElevenReader reads, skips or mangles; a measured words-per-hour rate; design choices updated | 1 evening |
| 1 | Extractor + debug viewer | `probe` and `extract` commands; an HTML page showing each PDF page with boxes coloured by role | Roles look right on 20 random pages of book #1 | 1 weekend |
| 2 | First listenable EPUB | Header/footer removal, dehyphenation, paragraph joins, chapters from bookmarks, metadata + cover, EPUB writer, EPUBCheck | EPUBCheck clean; a short excerpt of book #1 imports into ElevenReader with a correct TOC; book #2 under 1% character error | 2 weekends |
| 3 | Book furniture | Inline footnotes, sidebars, lists/definition lists, figures + captions, code cues, Unicode cleanup | Footnote pairing ≥ 98% on books #1 and #3; listening spot-checks pass | 2 weekends |
| 4 | Tables | Table detection, LLM summaries with number checking, rule-based fallback, table-as-image | Every table in book #1 has a summary that passes the check, or a fallback; \~20 min of CPU time | 1–2 weekends |
| 5 | Self-check | Narration lint, `report.html`, pass/warn/fail verdict, listening-time estimate, volume splitting by hours, snapshot tests over the test set | One command runs the whole test set and shows the regressions | 1 weekend |
| 6 | Desktop app | PySide6 window: drop a PDF, options, progress, open EPUB/report; PyInstaller build | A fresh Windows user account can install and convert book #1 with no terminal | 1–2 weekends |
| 7 | Offline audio | Kokoro narration per chapter → MP3s + M4B with chapters | Book #1 renders overnight; chapter marks match the TOC | 1 weekend |
| 8 | Hard layouts | Heading inference without bookmarks, two-column ordering, Docling fallback, OCR route, math | Books #4–#8 come out “warn” or better | Ongoing |

## Risks and open questions

The biggest risk is ElevenReader's undocumented EPUB handling. The cheapest fix is Milestone 0, which takes one evening.

| Risk | Impact | Mitigation |
| --- | --- | --- |
| ElevenReader reads or skips EPUB constructs differently than assumed (footnote markup, alt text, `<pre>`, hidden text) | Footnotes vanish, or tables and code get read aloud | Milestone 0 probe; keep spoken text as plain visible paragraphs, which every reader handles the same way |
| The 500-page limit counts EPUBs in a way that rejects this book | Import fails | Volume splitting (already in the MVP for the hours budget); test with a long dummy EPUB in Milestone 0 |
| Free hours run out (testing plus a ~15-hour book) | Can't finish a book, or test a release, that month | Hours estimate before import, 9-hour volumes, excerpt EPUBs for tests, a local voice for development listening, an hours log |
| ElevenReader changes the free plan | The budget math is off | The hours cap and narration pace are settings, not constants; re-check the pricing page before each release |
| No audio export from ElevenReader | Your MP3 plan doesn't work as stated | Stream in the ElevenReader app (online only on the free plan), and use the offline Kokoro export (Milestone 7) for MP3s. Don't record the app's audio output: its terms limit generated audio to in-app personal use |
| The LLM invents table facts | Wrong information read aloud | Number and name check against the cells; fallback sentence; summaries are labelled as summaries |
| Rules overfit to O'Reilly's typesetting | Other publishers' books degrade | Test set spans publishers; role thresholds come from each book's own font statistics, not fixed numbers |
| Fully automatic mode hides errors | You find problems an hour into listening | Lint verdict plus five targeted listening spots per book |
| CPU speed (LLM, OCR, Kokoro) | Long runs | Only tables use the LLM; OCR and audio run as resumable background jobs with per-chapter caching |
| Licensing if you share it | PyMuPDF and PyMuPDF Layout are AGPL; Marker's model weights are restricted | Fine for personal use; for a public release, open-source under AGPL or move to `pypdfium2` + Docling |
| Copyright of the books themselves | Converted books are for your own use | Don't share converted outputs; keep copyrighted test PDFs out of any public repo |

**Things I'm unsure about**

- How ElevenReader treats each EPUB construct; nothing is published, so Milestone 0 is how to find out.
- How ElevenReader counts “pages” in an EPUB, and whether this book needs splitting.
- Whether free hours reset weekly or monthly (ElevenLabs' pages say both), and the exact narration pace hours are charged at.
- Whether re-importing an edited EPUB counts as new text. It very likely does, so plan to import each book once.
- The speed figures for your PC: LLM tokens/s, OCR seconds per page and Kokoro characters per second are my estimates from other hardware. Benchmark each one in its milestone.
- Which small local model gives the most faithful table summaries. The models named are recent picks from one roundup, not something I've tested. Pick by trying two on the 46 tables in book #1.
- How hard EPUBCheck's Java requirement is to bundle in a PyInstaller build. A portable JRE adds \~40–50 MB; the alternative is a first-run prompt to install Java.
- Whether your planned Ryzen 7 5700G PC (also no GPU) changes the plan. It should only speed up the LLM, OCR and audio steps; the design doesn't depend on it.

## Sources

- ElevenLabs Help: [Can I download audio from ElevenReader?](https://help.elevenlabs.io/hc/en-us/articles/26165470379537-Can-I-download-audio-from-ElevenReader)
- ElevenLabs Help: [How do I add content to ElevenReader?](https://help.elevenlabs.io/hc/en-us/articles/26197616307985-How-do-I-add-content-to-ElevenReader)
- ElevenLabs Help: [What is the maximum file length supported in ElevenReader?](https://help.elevenlabs.io/hc/en-us/articles/44397769907217-What-is-the-maximum-file-length-supported-in-ElevenReader)
- ElevenLabs Help: [Which file formats can I import with Studio?](https://help.elevenlabs.io/hc/en-us/articles/25708839235345-Which-file-formats-can-I-import-with-Studio)
- ElevenLabs Help: [How do ElevenReader hours work?](https://help.elevenlabs.io/hc/en-us/articles/35971782968465-How-do-ElevenReader-hours-work)
- [ElevenReader pricing](https://elevenreader.io/pricing)
- [Introducing the all-new ElevenReader powered by Eleven v4](https://elevenreader.io/blog/introducing-elevenreader-v4) (updated Oct 2, 2026)
- [PyMuPDF on PyPI](https://pypi.org/project/pymupdf/) (2.1, Nov 2, 2026) and [pymupdf4llm on PyPI](https://pypi.org/project/pymupdf4llm/)
- [PyMuPDF Layout](https://openapps.pro/packages/pymupdf-layout)
- [Docling on GitHub](https://github.com/docling-project/docling)
- [Marker on GitHub](https://github.com/datalab-to/marker)
- [MinerU on GitHub](https://github.com/opendatalab/MinerU) and its [licence](https://github.com/opendatalab/MinerU/blob/master/LICENSE.md)
- [pdfmux: 200-PDF parser benchmark](https://pdfmux.com/blog/pdfmux-vs-pymupdf-vs-marker-vs-docling/) (Jun 2026)
- [Marker vs Docling 2026](https://aifoss.dev/blog/marker-vs-docling-foss-pdf-parsing-local-rag-2026/)
- [Best open-source PDF-to-Markdown tools in 2026](https://themenonlab.blog/blog/best-open-source-pdf-to-markdown-tools-2026)
- [Best CPU-only local LLMs in 2026](https://www.popularai.org/p/best-cpu-only-local-llm-2026) (Jul 2026)
- [audiblez (Kokoro-82M EPUB → M4B)](https://pypi.org/project/audiblez)
- [EPUBCheck releases](https://github.com/w3c/epubcheck/releases)
- Sample PDF analysis: PyMuPDF inspection of *AI Engineering* (your file), Nov 4, 2026
