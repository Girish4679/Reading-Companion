# Recap & Quiz

Two Week-2 AI features that turn a reading session into retained learning. Both are grounded in
material up to the current page and both degrade gracefully without a model (D9).

---

## Recap — `ctx read recap`

A short summary of what you just read/learned this session, so next time `resume` has substance.

**Flow**
1. Gather this session's inputs: the page range read, notes/takeaways made, concepts touched.
2. Retrieve the passages in the session's page range (locator-bounded).
3. `AIRequest(task=recap, model_hint=fast, effort=low)` — recaps are cheap (HLD §6 table).
4. Validate structured output; persist on the session (`reading_sessions.recap` +
   `recap_model_call_id`).

**Rules**
- **A session may end without a recap if the call fails** (HLD line 97) — never block finishing.
- Grounded: recap cites the pages it summarizes; ungrounded claims flagged.
- Short by design — a few sentences + the session's key takeaways, not a chapter summary.

---

## Quiz — `ctx read quiz [--n k]`

Generate recall questions from material up to the current page, to test understanding.

**Flow**
1. Select target concepts/passages up to current page (prefer recently-read and lower-understanding).
2. `AIRequest(task=quiz, model_hint=fast, effort=low)` → structured `Quiz { items[]: {question,
   answer, citations[]} }`.
3. Validate + grounding check (each item cites the passage it tests — D8).
4. Persist questions; present them for recall.

**Grading (OQ5)**
- **MVP: self-grade.** You answer from memory, then reveal the cited passage + model answer and mark
  yourself `correct / partial / incorrect`. Honest, cheap, no extra model call.
- Results write to `reviews` and update the concept's `understanding_level`.
- Model-assisted grading is a Next-phase option (OQ5).

**Rules**
- Questions only from material **at or before** the current page (no spoilers, consistent with
  concepts/explain).
- Never quiz `mastered` concepts (Week-3 exit criterion).
- Prefer concepts with unresolved questions or shaky understanding.

---

## Shared

- Both use the [AI adapter](../architecture/ai-adapter.md) with structured outputs + validation +
  one repair retry.
- Both log tokens/cost/latency (D10) and respect the "send only what's needed" budget.
- Both are **fast-tier, low-effort** by default — these are the routine, high-frequency calls the HLD
  specifically says not to overspend on.
- Quiz results feed [review](review.md)'s two streams and the retention metrics in
  [success-metrics](../product/success-metrics.md).
