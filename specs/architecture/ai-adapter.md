# AI Adapter

The small interface that keeps the app independent of any one model provider (HLD architectural
rule, line 47). Everything the app needs from "intelligence" goes through here.

## Interface (conceptual)

```
AIProvider
  name : str
  generate(request: AIRequest) -> AIResult
  capabilities() -> { supports_reasoning_effort, max_context, structured_output }

AIRequest
  task          : enum { explain, recap, quiz, extract_concepts, eval_grounding }
  system        : str                      # task-specific system prompt
  context[]     : ContextItem              # passages + notes, each with provenance
  output_schema : pydantic model           # expected structured shape
  model_hint    : enum { fast, mid, capable }   # maps to a concrete model per provider
  effort_hint   : enum { low, standard, high }  # provider maps to its own controls

AIResult
  ok            : bool
  data          : parsed+validated object | None
  raw_text      : str
  citations[]   : Locator refs the model claimed
  usage         : { input_tokens, output_tokens, model_id }
  cost_estimate : float
  latency_ms    : int
  error         : AIError | None            # typed: rate_limit, timeout, invalid_json, refusal, no_key
```

Features build an `AIRequest` and receive a validated `AIResult`. They never see provider SDK types.
This resolves **open question OQ1** with a concrete envelope.

## Provider adapters

- **ClaudeProvider** — default. `model_hint` → current Haiku/Sonnet/Opus-class ids; `effort_hint`
  → Anthropic's reasoning controls. See [quality/configuration-and-cost](../quality/configuration-and-cost.md)
  for the task→tier mapping and why we start at Sonnet.
- **OpenAIProvider** — alternative, same interface.
- **LocalProvider (Ollama)** — offline/stretch; may report reduced `capabilities()`.

Selection is config-driven (TOML). Swapping providers must not touch feature code.

## Structured outputs & validation

1. Each task declares a Pydantic `output_schema`.
2. The provider is asked for JSON matching that schema.
3. The adapter validates; on failure it **retries once** with a repair instruction, then returns
   `AIResult.ok = False` with `error = invalid_json`.
4. Features must handle `ok = False` without crashing (D9) — e.g. save the session anyway, mark the
   recap as unavailable.

## Grounding & citations

- `explain`, `recap`, and `quiz` requests include passages tagged with their `Locator`.
- The system prompt instructs the model to cite only from supplied passages and to return
  `citations[]`.
- The adapter runs a **grounding check**: every citation must resolve to a real supplied chunk, and
  explanations with zero valid citations are flagged `ungrounded`. See
  [quality/evaluation](../quality/evaluation.md). This enforces decision D8.

## Logging (decision D10)

Every call logs: task, model id, the **exact context sent** (passage ids + note ids, and full text
at debug level), token counts, cost estimate, latency, and ok/error. This makes "what left my
machine" inspectable and feeds the cost metrics. Stored in a `model_calls` log (see
[data-model](../data/data-model.md)).

## No-key / offline behavior (OQ1 default)

If no provider is configured or the network is unavailable, AI-dependent commands:
- print a clear message that the AI step was skipped,
- still perform their non-AI parts (e.g. `recap` saves the session; you can write the recap yourself),
- exit non-error so scripts/habits aren't broken.
Pure-core commands (`save`, `resume`, `progress`, `note`, `review` selection) are unaffected.

## Cost controls

- Default to the cheapest tier that passes eval for a task; never max effort by default (HLD §6).
- Send only what's needed: the user's question + relevant passages/notes up to the current page,
  **not** the whole book or full history (HLD line 175).
- Per-call and per-session cost is tracked and shown; a configurable soft budget can warn the user.
