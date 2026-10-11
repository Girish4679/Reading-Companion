# Explain

`ctx read explain <question>` — the primary "understanding" command. Answers a question about a
difficult concept using your question, your notes, and relevant book passages **up to your current
page**, and **cites the page/section** for every claim (D8).

## Why citations are mandatory (D8)
Two reasons: (1) evidence-based UX — you can verify against the paper book; (2) citations are the
ground truth for evaluation ([quality/evaluation](../quality/evaluation.md)). An explanation with no
valid citation is flagged, not presented as authoritative.

## Flow

```
question
  │ 1. retrieve: chunks relevant to the question, restricted to locator ≤ current page
  │              + the user's related notes (same concept/section)
  ▼
context (passages + notes, each tagged with its Locator)
  │ 2. build AIRequest(task=explain, output_schema=Explanation, model_hint=mid)
  ▼
AI provider  → structured Explanation { answer, claims[]: {text, citations[]} }
  │ 3. grounding check: every citation resolves to a supplied chunk
  ▼
persist: note(author=ai, type=explanation) + citations[] rows + model_call
  ▼
render: answer with inline page refs, e.g. "...replication lag (p. 162, §5.2)"
```

## Retrieval (MVP = Option A, D5)
- **No embeddings.** Select candidate chunks by: same section as the current/That concept, keyword/
  section match to the question, restricted to `locator ≤ current page` (never leak unread pages —
  consistent with the no-spoilers stance).
- Rank simply (section proximity + keyword overlap), take the top budget-fitting set.
- Include the user's related notes so the explanation accounts for "what you already understand"
  (HLD line 175).
- If retrieval quality proves too blunt, embeddings are the planned upgrade (Next phase) with no
  schema change — add an `embedding` column to `content_chunks`.

## Boundaries on what's sent
Send the question + selected passages + related notes only — **not** the whole book or full history
(HLD line 175). This keeps cost low (~3k input tokens target) and is logged (D10).

## Output: `Explanation` schema (conceptual)
```
Explanation
  answer    : str                 # the explanation prose
  claims[]  : { text: str, citations: [Locator] }
  caveats   : str | None          # e.g. "beyond your current page" refusal, low-OCR-confidence note
```

## Grounding & failure handling
- **Ungrounded:** zero valid citations → mark the note `ungrounded`, warn the user, still show the
  text but clearly labelled as not evidence-backed.
- **Out of range:** if answering well needs material past the current page, the model is instructed to
  say so rather than invent — respects pace and avoids spoilers.
- **Model/validation failure:** return gracefully (D9) — tell the user the explanation couldn't be
  generated; the question is still saved as an open `note` so it resurfaces in review.
- **Low OCR confidence:** if cited chunks come from low-confidence pages (ingestion flagged them), add
  a caveat so you double-check against the paper.

## Model tier
Mid-tier, standard reasoning by default (HLD §6 task table). Escalate effort only if evaluation shows
it measurably helps on your test cases — not by default.
