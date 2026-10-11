# Testing

pytest. The MVP's reliability promise — never lose progress, handle model failures gracefully — has
to be enforced by tests, not hope. This is a Week-3 must-have but tests are written alongside each
feature, not bolted on at the end.

## Priorities (what must be tested)

### 1. Persistence & state integrity (highest)
- `save --page` persists and `resume` restores the **exact** position and notes (the first-milestone
  guarantee).
- Atomic save: a simulated crash mid-save never loses the previously saved position.
- Re-ingestion idempotency: running `ingest` twice doesn't duplicate chunks.
- UUID/ULID keys, UTC timestamps, FK integrity.

### 2. Session transitions
- The state machine in [sessions-and-progress](../features/sessions-and-progress.md): start → save →
  finish; auto-close on new start; **invariant: at most one open session per source**.
- A session can end with a null recap (model failure path).

### 3. Model-failure paths (explicit HLD requirement)
Using a **fake AI provider** (no network in tests):
- rate_limit / timeout / invalid_json / refusal / no_key → command degrades gracefully, core data
  intact, exit code sane (D9/OQ1).
- invalid JSON triggers exactly one repair retry, then a clean `AIResult.ok = False`.
- `explain`/`recap`/`quiz` with no API key skip the AI step and still do their non-AI part.

### 4. Grounding & citations
- An explanation citing a non-supplied chunk is flagged `ungrounded`.
- Citations resolve to real `content_chunks` and render the correct **printed** page (offset applied).

### 5. Concept override rule (D6)
- Re-extraction never overwrites a `user` / `user_edited` concept.
- Concepts past the current page are **not** surfaced (no-spoiler gate).

### 6. Review selection (D7)
- `mastered` concepts are excluded (no re-quizzing completed material).
- Cumulative vs. chapter scoping returns the right sets.
- Unresolved-question concepts are prioritized.

## Test infrastructure

- **Fake/in-memory repository** and a **fake AI provider** so the whole app is testable offline,
  deterministically, with no API cost. (These are why the repository + AI interfaces exist.)
- **Fixture source:** a tiny synthetic "book" (a few pages of plain text with a known chapter/section
  map) so ingestion, retrieval, and citation tests don't depend on the real scanned PDF or OCR.
- **OCR is tested separately** on a couple of sample scanned pages (OQ2), not in the fast unit suite.

## What we don't over-test
No tests asserting exact AI prose (non-deterministic) — those belong in the
[evaluation](evaluation.md) curated set, scored on coverage/grounding, not string equality. Unit
tests cover structure, control flow, and failure handling; the eval set covers answer quality.

## CI readiness
The suite must run with no network and no API key (fakes only), so it's CI-friendly from day one —
which also keeps it green when the hosted migration adds a different repository implementation
(same interface, same tests).
