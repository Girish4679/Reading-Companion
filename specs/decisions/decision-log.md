# Decision Log

Chronological record of decisions made during requirements discovery, with rationale.
Each entry: the decision, why, and what it supersedes in the original HLD where relevant.

Legend: **[C]** confirmed by user · **[R]** my recommendation, accepted · **[A]** working assumption.

---

### D1 — Design for multiple sources now, implement incrementally **[C]**
Introduce the `Source` / `Locator` / `LearningSession` abstractions from day one, but build
only the physical-book (scanned PDF) and Markdown/text adapters in the MVP. PDFs-from-file,
web articles, and docs come later.
**Why:** The user wants to extend the project later (system-design material, docs, other books)
without a schema/interface rewrite. Low added cost now; avoids a painful migration.
**Supersedes:** The HLD's single-`books`-table model (HLD §4). Adopts the HLD appendix's
generalized `sources` direction.

### D2 — Ingest the full book text **[C]**
The system will ingest the book's text, mapped to page numbers, rather than relying only on
manually typed excerpts.
**Why:** The user wants the tool to "already know the concepts up to page N" and to produce
page-cited explanations. That is impossible from manual notes alone.
**Supersedes:** The HLD's founding premise of a text-free, physical-only workflow (HLD §3, line 49).

### D3 — Legitimate copy only; user supplies the file **[C]**
The book is the user's own legally-owned copy (physical 2nd edition, which they scanned to PDF).
The tool ingests a file the user provides; it never downloads or fetches book text from the internet.
**Why:** Copyright. We will not source pirated text. BYO-copy also fits the adapter model cleanly.

### D4 — Source is a scanned PDF; OCR quality is a known risk **[C/A]**
The user scanned their physical copy, so the input is images requiring OCR.
**Why / caveat:** OCR text and page mapping will have errors that bound citation accuracy.
Treated as a known risk to mitigate (see [ingestion-pipeline](../data/ingestion-pipeline.md)),
not a blocker. **[A]** that the scan is one PDF covering the book with recoverable page boundaries.

### D5 — Retrieval starts simple (Option A), embeddings deferred **[C]**
MVP uses structured text chunks + per-chapter concept lists queried with ordinary SQL and
page-range filters. Semantic/embedding retrieval is deferred to a later phase and added only
if `explain` demonstrably fails to find the right passages.
**Why:** Learning outcomes come from the workflow (recall, review), not retrieval sophistication.
Start cheap and simple; keep the chunk/locator model embedding-ready.

### D6 — Eager parse, lazy concept surfacing, user edits win **[C]**
Parse and chunk the whole PDF once at import (cheap, no model). Extract/surface *concepts* only
up to the user's current page, extracting further on demand if they jump ahead. User edits to a
concept always override regenerated values.
**Why:** Avoids spoilers from unread chapters; spreads model cost over the reading habit;
keeps the user in control (HLD principle: user-controlled).

### D7 — Two review streams **[C]**
Review supports a **cumulative** stream (everything read so far) and a **chapter-scoped** stream.

### D8 — Explanations must cite page/section **[C]**
Every AI explanation cites the page(s)/section(s) its claims are grounded in. Ungrounded output
is flagged. This is both an evidence-based UX requirement and the ground truth for evaluation.

### D9 — AI is optional; the core works offline **[C, from HLD]**
Progress tracking, saving, resuming, notes, and review scheduling must work with no API key and
no network. Only explain/recap/quiz/concept-extraction call a model.

### D10 — AI adapter logs outbound context **[R]**
"Local-first" means data is stored locally; it does not mean processing is local. Every model call
logs exactly what context (book passages, notes) was sent, so the user can see what left their machine.

### D11 — Design for a later hosted migration **[C]**
The MVP is local-first (single user, single machine). The architecture must not foreclose a later
move to a hosted deployment (e.g. Vercel + a managed Postgres-class DB). Persistence and config are
kept behind interfaces so the storage backend can change without rewriting features.
See [future-hosted-migration](../architecture/future-hosted-migration.md).

### D12 — No sync in the MVP **[A]**
Single machine, no multi-device sync, until the hosted migration. Correct me if you expect sync sooner.

---

*The following resolve the discovery open questions (OQ1–OQ6).*

### D13 — No-key behavior: skip gracefully **[C]** (OQ1)
When no API key is configured, AI commands skip the AI step with a clear message and exit 0, still
performing their non-AI part. Core commands are unaffected. Common response shape = `AIResult`.

### D14 — OCR: Tesseract, lightest config **[C]** (OQ2)
Use Tesseract in its lightest viable configuration. Known caveat: the input is a scanned (image) PDF,
so OCR error depends on scan quality — not error-free. Keep confidence-flagging + printed-page display
so text can be verified against paper. Calibrate on a sample page if scans prove noisy.

### D15 — Canonical locator = printed page number **[C]** (OQ3)
Trust the book's printed page numbers. Printed page is canonical for display and citation; PDF page
index is internal, with a stored offset.

### D16 — Concept granularity: section-level default **[C]** (OQ4)
Section-level concepts by default; users may add finer-grained concepts.

### D17 — Quiz grading: self-grade in MVP **[C]** (OQ5)
User self-grades recall (correct/partial/incorrect) after seeing the cited passage. Model-assisted
grading deferred to Next.

### D18 — Hosted target: multi-user, Vercel + Supabase **[C]** (OQ6)
Plan the hosted deployment for multiple users on Vercel + Supabase (Postgres), which adds auth and
per-user data partitioning. Remains a stretch deliverable; MVP only keeps the interfaces open for it.
Triggers residual items R1 (auth provider) and R2 (copyright posture when hosted) in
[open-questions](open-questions.md).
