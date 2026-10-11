# System Architecture

## Layers

```
┌─────────────────────────────────────────────────────────────┐
│ CLI (Typer)                                                   │
│   init · ingest · start · save · resume · progress            │
│   note · explain · recap · quiz · review · concepts · config  │
├─────────────────────────────────────────────────────────────┤
│ Application / workflow layer                                  │
│   session mgmt · progress tracking · concept surfacing        │
│   review selection · recap/quiz orchestration                 │
├───────────────┬───────────────────────────┬──────────────────┤
│ Source layer  │ Persistence layer          │ AI layer         │
│ adapters +    │ repository interface over  │ provider adapter │
│ normalized    │ SQLite (swappable backend) │ + prompt build + │
│ content model │                            │ validation + log │
├───────────────┴───────────────────────────┴──────────────────┤
│ SQLite (local)            │ External model provider (optional) │
│                           │ Claude · OpenAI · local            │
└───────────────────────────┴────────────────────────────────────┘
```

Arrows are dependency flow, not synchronous calls. **Most commands touch only the persistence
layer.** Only `explain`, `recap`, `quiz`, and concept extraction reach the AI layer.

## Key architectural rules

1. **The AI adapter is behind a small interface.** No feature imports a Claude- or OpenAI-specific
   request/response shape. Swapping providers is a config change. See [ai-adapter](ai-adapter.md).
2. **Persistence is behind a repository interface.** Features call `repo.save_session(...)`, never
   raw SQL inline. This is what makes the later hosted-DB migration a backend swap rather than a
   rewrite (D11). See [future-hosted-migration](future-hosted-migration.md).
3. **Sources are behind an adapter interface.** Each source type (physical book/scanned PDF,
   Markdown/text, later: PDF-from-file, URL) normalizes into one content model. See
   [source-abstraction](source-abstraction.md).
4. **The core runs with no model and no network.** The AI layer is optional at runtime; its absence
   degrades commands gracefully (D9), it never breaks save/resume.
5. **Every model call is logged** with what context was sent, tokens, cost, latency, model id (D10).

## Module boundaries (proposed package layout)

```
reading_companion/
  cli/              # Typer app, command definitions, output formatting
  app/              # workflow services: sessions, progress, concepts, review, recap, quiz
  sources/          # adapter interface + physical_book, markdown adapters; content model
  ingest/           # OCR, chunking, page mapping (used by sources)
  storage/          # repository interface + sqlite implementation + migrations
  ai/               # provider interface + claude/openai/local adapters, prompt builders, validation
  config/           # TOML loader, secrets resolution, provider selection
  eval/             # grounding checks, metrics, test-set harness
  models/           # Pydantic domain models shared across layers
```

Dependency direction: `cli → app → {sources, storage, ai, config}`; `app`, `sources`, `ai` all
depend on `models`. Nothing depends on `cli`. `storage` and `ai` know nothing about each other.

## Data flow examples

**`save --page 200` (no AI):** CLI → app.session.save_progress → storage.repo. Fast, offline, durable.

**`explain "why does replication lag matter?"`:** CLI → app orchestrator → (storage: fetch notes +
chunks up to current page) → ai.prompt_builder → ai.provider → ai.validation → persist explanation
as a note with citations + a cost/latency record → CLI renders answer with page references.

## Portability stance

Local-first today, but every cross-cutting dependency (DB, model provider, config/secrets, file I/O
for ingestion) sits behind an interface so a hosted deployment changes *implementations*, not
*features*. The concrete migration plan is in [future-hosted-migration](future-hosted-migration.md).
