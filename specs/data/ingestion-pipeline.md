# Ingestion Pipeline

Turns the user-supplied scanned PDF of DDIA into `sections` + `content_chunks` with printed-page
locators (D2, D4). Runs **eagerly once at import**; concept extraction is a *separate*, lazy step
(D6) covered in [features/concepts](../features/concepts.md).

## Inputs & legitimacy

- Input is a file the **user provides** — their own scan of a book they own (D3). The tool never
  downloads book text.
- The source is a **scan (images)**, so OCR is required (D4). This is the one source type that needs
  OCR; the Markdown adapter does not.

## Stages

```
PDF (scanned images)
  │  1. page split + preprocess   (deskew, denoise — light; heavy cleanup is stretch)
  ▼
per-page images
  │  2. OCR                       (text + per-page confidence)
  ▼
per-page text + PDF page index
  │  3. page-number mapping       (printed page ↔ PDF page; store offset)
  ▼
page-anchored text
  │  4. structure detection       (chapters/sections → sections table)
  ▼
sections
  │  5. chunking                  (split into retrieval/citation units, each keeps its locator)
  ▼
content_chunks  → persisted via the repository
```

### 1. Page split & preprocess
Split the PDF into page images. Light preprocessing only (deskew/denoise). Figure/equation handling
and aggressive cleanup are **stretch** (scope-and-roadmap), not MVP.

### 2. OCR (D14)
Run OCR per page, retaining a **confidence score**. Toolchain: **Tesseract, lightest viable config**,
run locally (keeps text on the user's machine). Note the input is a *scanned* (image) PDF, so OCR
error depends on scan quality and is **not** zero the way a born-digital PDF's embedded text would be.
Low-confidence pages are flagged in `sources.metadata` so citations drawn from them can be marked
lower-trust and verified against paper. Calibrate on a sample page if real scans are noisy.

### 3. Page-number mapping (D15)
Scanned books have front matter, so **PDF page N ≠ printed page N**. Resolved: the **printed** page
is canonical. We:
- detect or let the user specify the offset (e.g. "printed page 1 = PDF page 23"),
- store the offset in `sources.metadata`,
- make the **printed** page the canonical display/citation locator (what you'd find on paper),
- keep the PDF page internally for debugging only.

### 4. Structure detection
Detect chapter/section headings to populate `sections` with the hierarchy and page ranges. DDIA has
a clean chapter/section structure; detection can be heuristic (heading patterns) with a manual
fallback (user supplies a chapter→page map at `init`). A manual map is the reliable MVP path if
OCR headings are noisy.

### 5. Chunking
Split section text into `content_chunks` sized for prompting (a few hundred tokens each), each
stamped with its printed-page `locator` and `section_id`. Chunks are the unit `explain`/`recap`/
`quiz` cite (D8). Keep `token_estimate` per chunk for prompt budgeting.

## Idempotency & re-ingestion

- Re-running `ingest` on the same source must not duplicate chunks. Key chunks by
  (source_id, order_index) / content hash; re-ingest replaces cleanly.
- Ingestion is resumable: if it fails mid-way, re-running continues rather than corrupting state.
- Ingestion runs with **no AI** — it's OCR + parsing only. (Concept extraction, which is AI, is a
  later, separate, lazy step.)

## Quality risks (tracked)

| Risk | Mitigation |
|---|---|
| OCR errors in text → wrong citations | Store confidence; flag low-confidence pages; printed-page display so the user can verify against paper |
| Wrong printed↔PDF offset → citations point to wrong page | OQ3: capture offset explicitly; allow user override |
| Noisy heading detection → bad section structure | Manual chapter→page map fallback at `init` |
| Figures/equations as garbled text | Out of MVP scope; flag figure-heavy pages, don't rely on them |

## Output contract

After a successful ingest: the source has a populated `sections` tree and `content_chunks` covering
the book, each citable by printed page. `explain`/`recap`/`quiz` can now retrieve passages up to the
reader's current page by a simple locator comparison — no embeddings needed (D5).
