# Concepts

Concepts are the topics you're learning (e.g. "leader-based replication", "replication lag"). They
drive review and surface what you've covered. This feature implements the eager-parse / lazy-surface
/ user-override decision (D6).

## Lifecycle

```
ingest (eager)      →  text chunks + sections exist, NO concepts yet
reach page N        →  extract/surface concepts up to page N (lazy, AI)
user reviews list   →  edit / rename / merge / add / set understanding level
regeneration later  →  user edits always win (origin = user / user_edited)
```

### Eager vs. lazy — why this split (D6)
- **Eager** on *text* (ingestion): cheap, no model, needed for retrieval.
- **Lazy** on *concepts*: a concept is only surfaced once your current page reaches its
  `first_locator`. This avoids **spoilers** from chapters you haven't read, spreads model cost across
  the reading habit, and matches your pace. Jumping ahead triggers on-demand extraction for the
  newly-covered range.

See the pros/cons table in [decision-log D6](../decisions/decision-log.md).

## Extraction
- Trigger: advancing your saved page past concepts not yet extracted, or an explicit
  `ctx read concepts refresh`.
- Input to the model: the chunks in the newly-covered page range (not the whole book). Task =
  `extract_concepts` via the [AI adapter](../architecture/ai-adapter.md), structured output validated.
- Each extracted concept gets: `name`, `section_id`, `first_locator`, `understanding_level = seen`,
  `origin = generated`.
- Granularity (section-level vs. finer) is **OQ4**; default section-level, user may add finer ones.

## User control (D6 — overrides win)
- `ctx read concepts list` — show concepts up to current page, with understanding levels.
- `ctx read concepts edit <id>` — rename/redefine; sets `origin = user_edited`.
- `ctx read concepts add <name>` — manual concept; `origin = user`.
- `ctx read concepts level <id> <unseen|seen|shaky|solid|mastered>` — set understanding.
- **Regeneration rule:** re-extraction never overwrites a concept whose `origin` is `user` or
  `user_edited`. It may add new generated concepts and update `generated` ones. This is the
  "user-controlled" principle made concrete.

## Understanding level
Progression: `unseen → seen → shaky → solid → mastered`. Updated by:
- review results ([review](review.md)) — correct answers promote, incorrect demote,
- explicit user setting.
Used to (a) pick what's due for review and (b) **exclude mastered concepts** so you're never
re-quizzed on completed material (Week-3 exit criterion).

## Relationship to notes and explanations
- A `note` or AI `explanation` can be linked to a concept (`notes.concept_id`), building up the
  evidence around it.
- Unresolved questions tied to a concept surface it as "needs attention" in review.

## Deliberately deferred
No knowledge graph / concept-to-concept links in the MVP (stretch). The `concepts` table is the seed
a graph would later connect, but MVP keeps them a flat, per-source list.
