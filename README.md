# Reading Companion — Specifications

Context Engine · Reading Mode. A local-first CLI that helps you read, understand, and
retain technical material — starting with *Designing Data-Intensive Applications, 2nd ed.*
(DDIA), designed to extend to other sources and, later, a hosted deployment.

These specs are the output of the requirements-discovery phase. They are written to be
implemented from, not to be code. Each spec separates **Confirmed** requirements from
**Assumptions** (things I inferred and will proceed on unless corrected) and
**Recommendations** (my proposed approach, open to change).

## How the specs are organized

| Folder | What's in it |
|---|---|
| [`decisions/`](decisions/) | The decision log from discovery, and still-open questions. Read this first. |
| [`product/`](product/) | Vision, goals, principles, scope/roadmap, success metrics. |
| [`architecture/`](architecture/) | System layers, the source abstraction, the AI adapter, and the future hosted migration. |
| [`data/`](data/) | The SQLite data model and the ingestion pipeline (scanned PDF → chunks). |
| [`features/`](features/) | One spec per capability: CLI, sessions, concepts, explain, recap/quiz, review. |
| [`quality/`](quality/) | Evaluation, testing, configuration, and cost/model policy. |

## Reading order for a new contributor

1. [product/overview.md](product/overview.md) — what this is and why.
2. [decisions/decision-log.md](decisions/decision-log.md) — the choices already made and their rationale.
3. [architecture/system-architecture.md](architecture/system-architecture.md) — the shape of the system.
4. [data/data-model.md](data/data-model.md) — the persistent state everything hangs off.
5. The `features/` specs, in any order.

## Status

Discovery complete. These are **v1 draft specs** pending your review. Nothing here is built yet.
Open items that still need your input are tracked in
[decisions/open-questions.md](decisions/open-questions.md).

## Core stack (confirmed)

Python · Typer (CLI) · SQLite · Pydantic (validation) · pytest · Claude Sonnet as the
default model behind a provider interface · TOML configuration · installable local CLI.
