# Source Abstraction

The abstraction that lets the project grow beyond one book without rewriting sessions, storage, or
AI integration (decision D1). Designed now; only two adapters built in the MVP.

## The three core abstractions

- **Source** — the thing you learn from: DDIA (scanned PDF), a Markdown file, later a web article.
  Has metadata (title, type, URI, edition) and structure (sections).
- **Locator** — *where* you are, in terms native to the source: a printed page number for a book,
  a PDF page for a file, a heading/anchor for an article, a line/section for Markdown. A locator is
  comparable within a source so "up to here" is well-defined.
- **LearningSession** — *what you did*: what you read, understood, and want to revisit. Source-agnostic.

## Normalized content model

Every adapter translates its source into the same shape, so the rest of the app only ever sees this:

```
NormalizedContent
  source_metadata : { title, source_type, uri, edition, extra }
  sections[]      : { id, parent_id, title, order, start_locator, end_locator }
  chunks[]        : { id, section_id, text, locator, order }
  locator_scheme  : how to parse/compare/display this source's locators
```

- **Sections** carry the heading hierarchy (chapter → section → subsection).
- **Chunks** are retrieval/citation units of text, each stamped with the locator it came from so an
  explanation can cite "p. 147, §5.2".
- **locator_scheme** knows how to render a locator for humans (printed page) and order chunks.

## Adapter interface (conceptual)

```
SourceAdapter
  can_handle(input) -> bool
  ingest(input, config) -> NormalizedContent      # may run OCR, parse, chunk
  make_locator(raw) -> Locator                      # "200" -> printed-page locator
  format_locator(locator) -> str                    # for display/citation
```

Adapters are pure translators: they produce `NormalizedContent` and know nothing about SQLite,
sessions, or the AI layer.

## MVP adapters

### PhysicalBookAdapter (scanned PDF) — primary
Input: the user-supplied scanned PDF of DDIA. Runs the ingestion pipeline (OCR → chunks → page map).
Locator = **printed** page number (with a PDF-page offset stored; see
[open-questions OQ3](../decisions/open-questions.md)). This is the only source that needs OCR.
Details in [ingestion-pipeline](../data/ingestion-pipeline.md).

### MarkdownTextAdapter — secondary
Input: a `.md`/`.txt` file (e.g. your own notes, or docs). Locator = section heading + line range.
No OCR; chunking by headings/paragraphs. Proves the abstraction with a trivially legal, simple source.

## Deferred adapters (design must not block them)

- **PdfFileAdapter** — born-digital PDFs/papers via a text parser (e.g. PyMuPDF), locator = PDF page.
- **UrlArticleAdapter** — extract main article body where permitted, retain source URL; locator = heading.

Adding one of these must require only: a new adapter class + registration. No schema change beyond
`source_type`/`locator` values the model already allows (see [data-model](../data/data-model.md)).

## Why this is worth the small upfront cost

The HLD appendix called this out: separating *what* you're reading from *how* it's accessed means
session management, the database workflow, and AI integration are written once and reused for every
future source. Confirmed as the baseline in D1.
