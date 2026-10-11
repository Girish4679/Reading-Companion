# Open Questions

Discovery resolved all six. Resolutions are recorded as decisions D13–D18 in the
[decision-log](decision-log.md); this file keeps the question → answer trail.

### OQ1 — No-key / AI-adapter behavior → **RESOLVED (D13)**
Delegated to me. Decision: when no API key is configured, AI commands **skip the AI step with a
clear message and exit 0**, still doing their non-AI part; pure-core commands are unaffected. The
common response envelope is the `AIResult` type in [ai-adapter](../architecture/ai-adapter.md).

### OQ2 — OCR toolchain & error expectation → **RESOLVED (D14)**
**Tesseract, lightest viable config.** Caveat recorded: the source is a *scanned* PDF (images), so
OCR error depends on scan quality — it is **not** error-free the way a born-digital PDF would be.
Confidence-flagging and printed-page display stay, so low-confidence text can be verified against
paper. We may still calibrate on a sample page if real scans are noisy.

### OQ3 — Page-number source of truth → **RESOLVED (D15)**
**Trust the book's printed page numbers.** The printed page is canonical for display and citations;
the PDF page index is internal only, with a stored offset mapping one to the other.

### OQ4 — Concept granularity → **RESOLVED (D16)**
**Section-level by default**, with finer user-added concepts allowed.

### OQ5 — Quiz/recall grading → **RESOLVED (D17)**
**Self-grade in the MVP** (you mark correct/partial/incorrect after revealing the cited passage).
Model-assisted grading is a Next-phase option.

### OQ6 — Hosted migration target → **RESOLVED (D18)**
**Plan for multiple users.** Target: Vercel + Supabase (Postgres). Multi-user implies **auth** and
per-user data partitioning, plus a privacy review for ingested copyrighted text leaving the user's
machine. Still a stretch deliverable — not built in the MVP — but the data model and storage
interface are shaped to allow it. See [future-hosted-migration](../architecture/future-hosted-migration.md).

---

## New residual items (surfaced by the OQ6 answer)

These are not blockers for the MVP but should be decided before the hosted migration:

- **R1 — Auth provider.** Supabase Auth is the natural default given Supabase Postgres. Confirm when
  we get there.
- **R2 — Copyright posture when hosted.** Storing a user's scanned copyrighted book text on a shared
  server (vs. their own machine) is a materially different legal posture than local-first. Must be
  revisited before hosting ingested book content — likely each user ingests their **own** copy and
  content is partitioned and private per user.
