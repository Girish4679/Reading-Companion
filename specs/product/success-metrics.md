# Success Metrics

Success is **behavior change and retention**, not how impressive the AI output looks. Metrics below
are instrumented by the app where possible (marked 📊) or assessed periodically by the reader (🧑).

## Primary metrics

| Metric | Target (first 4 weeks) | How measured |
|---|---|---|
| 📊 Reading sessions per week | ≥ 3 | Count of completed sessions |
| 📊 Time to resume after a break | Trends down; feels instant | Time from `resume` to first action |
| 📊 Position restored correctly | 100% | Saved vs. restored page/chapter/section match |
| 📊 Sessions with a saved takeaway | ≥ 80% | Sessions that persisted ≥1 note of type takeaway |
| 🧑 Recall accuracy on prior concepts | Improving over time | Review results (right/wrong) trend |
| 📊 AI cost per completed session | < a few cents | Sum of per-call cost in session window |

## Guardrail metrics (watch for harm)

| Metric | Threshold | Action if breached |
|---|---|---|
| 📊 Ungrounded explanations | Near 0 | Explanation lacking citations is flagged, not shown as authoritative |
| 🧑 Incorrect explanations/quizzes | Low | Logged to the eval set; tighten prompts or lower-trust that model tier |
| 📊 Re-quizzing completed material | 0 | Review selection must exclude mastered concepts |
| 📊 Progress-loss incidents | 0 | Any lost position is a release blocker |

## The headline target (from the HLD)

Three reading sessions per week for four weeks, with the tool restoring position correctly every
time. Then assess whether you're remembering more than you would by reading and highlighting alone.

## Relationship to evaluation

These metrics are *product* outcomes. The mechanics of grounding checks, the curated test set, and
per-call quality scoring live in [quality/evaluation.md](../quality/evaluation.md).
