# CLI

The command surface (Typer). Commands split into **core** (no AI, offline, fast) and **AI** (call a
model; degrade gracefully when no key — D9/OQ1). Command names below are proposed; the HLD's were
illustrative.

The HLD used a `ctx read ...` namespace, anticipating this becoming one module of a larger Context
Engine. We keep that: the app installs as `ctx` with a `read` sub-app, so `ctx start`, `ctx explain`,
etc. The examples below show the `read` sub-app.

## Core commands (no AI required)

| Command | Purpose |
|---|---|
| `ctx read init <title> [--edition] [--file scan.pdf] [--chapters map.toml]` | Register a source; optionally kick off ingestion and supply a chapter→page map. |
| `ctx read ingest [--file scan.pdf]` | Run/resume the ingestion pipeline for the current source. OCR + parse only. |
| `ctx read start` | Begin a reading session from the last saved position. |
| `ctx read save --page <n> [--section <id>]` | Save current position. **Must work offline, instantly** (HLD line 71). |
| `ctx read resume` | Show where to resume: last position, last session's takeaways, suggested next steps. |
| `ctx read progress` | Show overall progress: current page/chapter, % through, sessions, open questions. |
| `ctx read note <text> [--type takeaway\|question\|excerpt] [--concept <id>]` | Record a note. |
| `ctx read concepts [list\|edit\|add\|level] ...` | View/override concepts up to current page (D6). |
| `ctx read review [--stream cumulative\|chapter] [--chapter <id>]` | Run a review session (two streams, D7). |
| `ctx read config ...` | View/set provider, model tiers, budgets, keys location. |

## AI commands (degrade gracefully without a key)

| Command | Purpose |
|---|---|
| `ctx read explain <question>` | Explain a concept using the question + relevant passages up to current page, **with page citations** (D8). |
| `ctx read recap` | Generate a short recap of the current/just-ended session. |
| `ctx read quiz [--n <k>]` | Generate recall questions from material up to current page. |

## Cross-cutting CLI behavior

- **Current source is implicit.** A "current source" (last `init`/used) avoids repeating it. `--source`
  overrides.
- **Offline-safe exit codes.** Core commands never fail due to missing AI/network. AI commands print a
  clear "AI step skipped" message and exit 0 when no key is configured (OQ1).
- **Output is human-first but scriptable.** Readable tables/prose by default; `--json` for machine output
  (useful later for the Context Engine integration and the hosted API).
- **Everything persists through the repository** — no command writes SQL inline (D11).
- **Cost visibility.** AI commands print tokens + estimated cost; `progress` shows cost per session.

## Resume output (the headline UX, from HLD §3)

`ctx read resume` should reproduce something like:

```
DDIA · Chapter 5 · Replication
Last position: page 147
Last session: read 12 pages, reviewed leader-based replication,
              left 1 open question about replication lag.

Next session
  • Review the previous session's 3 takeaways.
  • Read the next 5–10 pages, or to the next section break.
  • Note one confusing concept; `ctx read explain` it.
  • Answer 2 recall questions without looking.
  • `ctx read save --page <n>` with your new page + open questions.
```

This is assembled entirely from local data (sessions, notes, progress) — **no AI call** — so `resume`
is instant and works offline.

## Not in the MVP

Interactive TUI, shell completion niceties beyond Typer defaults, and any web/HTTP surface (that
arrives with the hosted migration, reusing the same `app/` services).
