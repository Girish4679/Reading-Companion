# Sessions & Progress

The core, AI-free workflow — the part that must be fast and dependable even offline (HLD line 71).
This is the Week-1 heart of the MVP and the first milestone's load-bearing feature.

## Concepts

- **Reading session**: a bounded period of reading, with a start/end position and the notes you made.
- **Progress**: the current position in a source (page, section, chapter) — the thing `resume` restores.

## `start`
1. Load the current source and its last saved progress.
2. Open a new `reading_sessions` row (`started_at` now, `start_locator` = current position,
   `ended_at` null).
3. Print the `resume` view (below) so you begin knowing exactly where you are.
   - If a session is already open, either continue it or warn — don't silently create a second.

## `save --page <n> [--section <id>]`
**The reliability keystone.** Must:
- Update the open session's `end_locator` and the source's `progress` (single fast write).
- Work with **no AI and no network** (HLD line 71). Never blocked by a model call.
- Be atomic — a crash mid-save must not lose the prior saved position.
- Optionally accept open questions inline (or via `note --type question`).

Saving does **not** require an open session (you can `save` a position ad hoc); if one is open it
updates it.

## `resume`
Reconstructs context from local data only (no AI):
- current position (page / chapter / section),
- the last session's summary line (pages read, concepts touched, open questions left),
- suggested next steps (review last takeaways, read N pages, note a concept, answer recall Qs, save).

The exact layout is in [cli.md](cli.md). The point: **zero rebuild cost** when you sit back down.

## `progress`
A read-only overview: current chapter/page, approximate % through (from section page ranges),
sessions this week, count of open/unresolved questions, and AI cost so far. Pure SQL aggregation.

## Finishing a session
A session ends when you `save` and explicitly finish, or when you `start` a new one (the previous is
auto-closed). On finish:
- set `ended_at`,
- optionally generate a `recap` (AI — [recap-and-quiz](recap-and-quiz.md)); **the session must be
  allowed to end even if the recap call fails** (HLD line 97, `recap` nullable in the schema).

## State transitions

```
          start                save --page           finish / start-next
no session ─────▶ in-progress ───────────▶ in-progress ──────────────▶ closed
                      ▲                          │
                      └──────── save (update) ───┘
```

Invariant: **at most one open session per source.** Transitions are unit-tested
([quality/testing](../quality/testing.md)), including the crash-during-save and model-failure paths.

## Offline guarantee (explicit)

Every command in this spec runs fully offline with no API key. This is the "small core" principle and
the first-milestone exit criterion: close the terminal, come back tomorrow, recover exact position and
notes — guaranteed, no intelligence required.
