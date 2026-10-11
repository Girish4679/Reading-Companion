# Future Hosted Migration

Decision D11: the MVP is local-first, but we must not make choices that force a rewrite when we
later host it (the user mentioned Vercel + a managed database). This spec records what we do **now**
so that migration is an implementation swap, not a redesign. It does **not** schedule the migration —
that remains a stretch deliverable (see [scope-and-roadmap](../product/scope-and-roadmap.md)).

## What we do now to stay portable

1. **Repository interface over storage.** All persistence goes through a `Repository` interface
   (`save_session`, `get_resume_state`, `list_concepts_up_to`, …). The MVP implements it with SQLite.
   A hosted deployment implements the same interface against a managed DB. No feature imports `sqlite3`.
2. **SQL kept portable.** Use ANSI-ish SQL and avoid SQLite-only idioms where a cheap alternative
   exists; isolate any SQLite-specific bits in the sqlite implementation. Timestamps stored as UTC ISO-8601.
3. **IDs that survive a move.** Use application-generated UUIDs (or ULIDs) as primary keys rather than
   relying on SQLite rowids, so rows can be exported/imported without key collisions.
4. **Migrations from day one.** A migrations mechanism (even if minimal) so schema can evolve and be
   replayed on a different backend.
5. **Config & secrets behind an interface.** TOML + env vars locally; the same `config` surface can be
   fed by platform env vars (Vercel) later. No secrets in source (HLD line 246).
6. **No local-filesystem assumptions in the core.** Ingestion writes through a storage abstraction so
   extracted text/chunks can live in a DB/object store rather than only local files.
7. **Stateless services where practical.** Workflow services take state in and return results, rather
   than holding long-lived local handles — friendlier to a serverless runtime.

## What changes at migration time (and what doesn't)

| Concern | Local (MVP) | Hosted (future) | Rewrite? |
|---|---|---|---|
| Storage backend | SQLite file | Supabase (Postgres) | No — new `Repository` impl |
| Ingested text/chunks | Local files + DB | DB / object store | No — storage abstraction |
| Model calls | Direct from CLI process | API route / serverless fn | No — same `AIProvider` |
| Config/secrets | TOML + env | Platform env vars | No — same `config` surface |
| Entry point | CLI (Typer) | HTTP API + optional web UI | Partially — new transport, same `app/` services |
| Auth / multi-user | none (single user) | multi-user via Supabase Auth | **New** — see R1/R2 |

The deliberate payoff: the `app/` workflow services and `models/` are reused as-is; only transport,
storage impl, and (if multi-user) auth are new.

## Confirmed target (D18)

**Multi-user, Vercel + Supabase (Postgres).** This makes auth and per-user data partitioning
first-class concerns at migration time (not the MVP). Implications to prepare for:
- A `users` table and `user_id` FKs on all user-owned rows (sources, sessions, notes, concepts,
  reviews, model_calls). The MVP schema is single-user; adding user scoping is additive.
- Auth: **Supabase Auth** is the natural default (residual R1 — confirm at migration time).
- Row-level security / per-user partitioning so one user's data (and ingested book) is private.

## Open items before any migration

- **R1 — auth provider**: Supabase Auth assumed; confirm when scheduled.
- **R2 — copyright posture when hosted**: storing a user's scanned copyrighted book on a shared
  server is a materially different legal posture than local-first. Each user must ingest their **own**
  legally-owned copy, stored private and partitioned to them. Revisit before hosting ingested book text.

## Explicit non-commitment

We are **not** adopting a client/server split, an ORM, or Postgres in the MVP just because hosting is
possible later. That would be premature. We adopt only the cheap interface boundaries above; anything
heavier waits until the migration is actually scheduled.
