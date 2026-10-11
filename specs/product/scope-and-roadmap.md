# Scope & Roadmap

A three-week MVP target, useful by the end of week one. Phases below refine the HLD's plan with
the discovery decisions (text ingestion, multi-source abstractions, two review streams).

## MVP (weeks 1–3)

### Week 1 — Core reading workflow (no AI required)
**Must-have**
- Project scaffold: installable CLI (Typer) + SQLite schema (Pydantic models).
- Initialize a source with its metadata and chapter/section structure.
- Ingest the scanned PDF: OCR → text chunks mapped to printed page numbers (eager parse). See
  [ingestion-pipeline](../data/ingestion-pipeline.md).
- Save and restore current page, chapter, and section.
- Start and finish sessions; record takeaways and open questions as notes.
- `resume` the last session in one command.

**Exit criterion:** close the terminal, return the next day, and recover your exact saved position
and notes — offline, no API key.

### Week 2 — AI reading companion
**Must-have**
- `explain` a concept using your question + relevant ingested passages (up to your current page),
  with page/section **citations**.
- Generate a short session **recap**.
- Generate recall **questions** from material up to your current page.
- **Concept extraction** (lazy, up to current page) with user edit/override.
- Structured-response validation and graceful API-error handling.
- Track token usage, cost, and latency per call.

**Exit criterion:** complete a session and save a useful, grounded recap and quiz without writing
them by hand.

### Week 3 — Learning & reliability
**Must-have**
- **Review** on two streams: cumulative and chapter-scoped.
- Surface unresolved questions from earlier chapters.
- Unit tests for persistence, session transitions, and model-failure paths.
- Evaluation: measure whether recaps/quizzes are useful and whether explanations are grounded.
- Configuration for swapping AI providers.

**Exit criterion:** reliably supports a multi-session habit; never loses progress; doesn't re-quiz
completed material.

## Next (post-MVP)

- Embedding/semantic retrieval for `explain` (Option B), if SQL/page-range retrieval proves too blunt.
- Additional source adapters: arbitrary PDFs, web articles (URL import), Markdown/docs sets.
- Spaced-repetition scheduling (promote review from "due by rule" to interval-based).
- Model-assisted quiz grading.

## Stretch

- Scanned-figure/equation understanding; OCR cleanup beyond text.
- Cross-document knowledge graph (replication ↔ partitioning ↔ transactions ↔ consistency).
- Cross-book memory; browser/voice capture; progress analytics.
- **Hosted migration** (Vercel + managed DB). Architecturally prepared from day one (D11), but the
  actual deployment is a stretch deliverable. See [future-hosted-migration](../architecture/future-hosted-migration.md).

## Non-goals (MVP)

Autonomous agents, multi-agent orchestration, a web dashboard, multi-user/sync, and fetching book
text from the internet (we only ingest the user-supplied file).
