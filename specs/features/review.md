# Review

The Week-3 "remembering" feature. Selects concepts due for review and tests recall. Supports the
**two streams** you asked for (D7): cumulative and chapter-scoped.

## The two streams (D7)

| Stream | Scope | Use |
|---|---|---|
| **Cumulative** | All concepts across everything read so far (up to current page) | Guard against forgetting earlier material as you progress |
| **Chapter** | Concepts within one chapter/section | Consolidate a chapter you just finished before moving on |

`ctx read review --stream cumulative` or `--stream chapter --chapter <id>`.

## Selection logic (MVP — rule-based, not yet spaced repetition)

A concept is **due** if it's within the chosen stream's scope AND:
- `understanding_level` is `seen`, `shaky`, or `solid` (never `mastered` → **excludes completed
  material**, Week-3 exit criterion), AND
- it hasn't been reviewed recently (`last_reviewed_at` older than a simple threshold), OR
- it has an **unresolved question** attached (prioritized).

Ordering: unresolved-question concepts first, then lowest understanding, then least-recently-reviewed.
Cap the batch size so a review session is finishable.

> This is intentionally a simple rule. **Spaced repetition** (interval scheduling based on recall
> performance) is the planned upgrade (Next/stretch). The `reviews` history and `understanding_level`
> are exactly the inputs a spaced-repetition scheduler needs, so the upgrade is additive.

## Review flow
1. Select due concepts for the chosen stream.
2. For each, present a recall prompt. Questions come from stored quiz items where available, else
   generate on the fly ([recap-and-quiz](recap-and-quiz.md)) — or run offline with user-written recall
   if no AI.
3. User answers from memory; reveal cited passage + mark `correct / partial / incorrect`.
4. Write a `reviews` row (with `stream`), update `understanding_level` (correct promotes, incorrect
   demotes), set `last_reviewed_at`.

## Unresolved questions from earlier chapters
A distinct Week-3 must-have: `review` (and `progress`) surfaces open `note`s of type `question`
(`resolved = false`) from earlier chapters, so questions you left behind don't vanish. Answering one
in review (or via `explain`) can mark it resolved.

## Offline behavior
Review **selection** is pure SQL and works offline. Only on-the-fly *question generation* needs AI;
without it, you review using stored questions or free recall, and still record results. Consistent
with D9.

## Metrics tie-in
Review results are the raw material for **recall accuracy over time** and the "not re-quizzing
completed material" guardrail in [success-metrics](../product/success-metrics.md).
