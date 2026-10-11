# Configuration & Cost

How the tool is configured (provider swap, model tiers, budgets, secrets) and how AI cost is kept
trivial. Configuration is TOML + env vars, behind a `config` surface that a hosted deployment can
later feed from platform env vars (D11).

## Configuration (TOML)

A local config file (e.g. `~/.config/ctx/config.toml` or project-local), plus env vars for secrets.
Illustrative shape:

```toml
[provider]
name = "claude"              # claude | openai | local
# model tiers map abstract hints -> concrete model ids
[provider.models]
fast    = "claude-haiku-5.5"
mid     = "claude-sonnet-5.5"
capable = "claude-opus-5.5"

[effort]
default = "standard"         # low | standard | high — raise only if eval justifies (HLD §6)

[budget]
warn_per_session_usd = 0.25  # soft warning, not a hard stop
show_cost = true

[ingest]
ocr_engine = "ocrmypdf"      # OQ2
printed_page_offset = 22     # OQ3 — printed page 1 = PDF page 23
```

- **Keys are never in config or source** (HLD line 246). API keys come from env vars (e.g.
  `ANTHROPIC_API_KEY`) or a local secret manager.
- Switching providers is editing `[provider]` — no code change (the AI adapter contract,
  [ai-adapter](../architecture/ai-adapter.md)).
- `ctx read config ...` reads/writes these values.

## Model & effort policy (from HLD §6)

| Task | Tier | Effort |
|---|---|---|
| Session recap | fast | low |
| Flashcards / basic quiz | fast | low |
| Explain a concept | mid | standard |
| Compare concepts / debug a misunderstanding | mid | standard, raise only if needed |
| Evaluate a recap's accuracy | capable (occasionally) or manual test set | — |

Start at Claude Sonnet as the default `mid` tier (HLD "starting choice"). Capability and reasoning
effort are distinct knobs; default to standard and let [evaluation](evaluation.md) justify any
increase.

## Cost model & tracking

- Every call logs tokens, model, estimated cost, latency to `model_calls` (D10).
- Planning estimate (HLD §6): ~3,000 input + ~800 output tokens per typical request. At Sonnet's
  listed rates that's roughly a cent and a half per call; realistic monthly use lands around a few
  dollars at most. **These are planning estimates, not quotes** — verify current provider pricing
  before implementation, and note a Claude *subscription* is billed separately from Claude *API*
  (Console) access (HLD §6).
- Cost containment levers already in the design: send only question + relevant passages + related
  notes (never the whole book/history), prefer the cheapest tier that passes eval, cache-friendly
  prompt construction, and a soft per-session budget warning.

## Secrets & privacy

- Keys in env/secret manager only.
- Every model call logs *what context was sent* (D10) so "what left my machine" is auditable — this
  matters because ingested book passages are copyrighted text going to a third-party API.
- Local model (Ollama) is the offline/privacy option (stretch) when you want nothing to leave the
  machine.

## Resolved config decisions
- OCR engine: **Tesseract, lightest config** (D14).
- Printed-page offset is canonical; store the offset (D15).
- No-key default behavior: **skip the AI step with a message, exit 0** (D13).
