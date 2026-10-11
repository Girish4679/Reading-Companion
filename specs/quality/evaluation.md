# Evaluation

How we judge whether the tool *works* — meaning it changes behavior and aids retention, not whether
the AI output looks impressive (HLD §7). Two halves: **product metrics** (do you read/retain more)
and **output-quality checks** (are AI answers correct and grounded).

## 1. Grounding checks (automated, every AI call)

Enforces D8. For `explain`/`recap`/`quiz`:
- Each claimed citation must resolve to a chunk actually supplied in the request context.
- An item with zero valid citations is flagged `ungrounded` and surfaced as such to the user.
- Grounding pass/fail rate is tracked over time — the "ungrounded explanations ≈ 0" guardrail
  ([success-metrics](../product/success-metrics.md)).

This is cheap, deterministic, and runs without a second model call. It's the backbone of
"evidence-based," made possible precisely because we ingested the book text (D2).

## 2. Curated test set (the quality ground truth)

A small, hand-maintained set of cases — resolving the "what is correct?" question now that we have
book text to check against:
- **Explain cases:** a question + the expected key points + the pages that support them. Score an
  explanation on coverage of key points and citation correctness.
- **Quiz cases:** a passage + acceptable answers, to sanity-check generated questions aren't
  nonsensical or ungrounded.
- **Recap cases:** a page range + the facts a good recap must include.

Run the set:
- when changing prompts, model, or reasoning effort (regression guard),
- periodically against real sessions.

Occasionally a more capable model can grade the set (HLD §7 "evaluate whether a recap is accurate"),
but a manually-reviewed set is the dependable baseline.

## 3. Product/behavior metrics

Instrumented from the DB (sessions, notes, reviews, model_calls) — defined in
[success-metrics](../product/success-metrics.md): sessions/week, time-to-resume, takeaway rate,
recall accuracy trend, cost/session. These answer "is the habit forming and is retention improving?"

## 4. What we explicitly measure about AI cost

From the `model_calls` log (D10): tokens, cost, latency per task and per session. Lets us verify the
HLD's claim that AI cost stays trivial (~cents/month) with focused prompts, and catch regressions
(e.g. a prompt that accidentally sends the whole book).

## Reasoning-effort policy (HLD §6, line 174)

Start every task at the **default/standard** effort for its tier. Increase effort only when the test
set demonstrates measurably better answers for that task. Capability (model) and effort are separate
knobs; don't raise either by reflex.

## Non-goals
No automated LLM-as-judge pipeline as the *primary* quality gate in the MVP (the curated set +
grounding checks are enough and cheaper). No benchmark theater — every metric must map to "read more"
or "remember more."
