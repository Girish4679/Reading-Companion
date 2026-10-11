# Product Overview

## What it is

A local-first command-line **reading companion** that helps a single reader make steady progress
through dense technical material, understand hard concepts, and retain them across many sessions —
without rebuilding context each time they sit down. The first supported source is the reader's own
scanned copy of *Designing Data-Intensive Applications, 2nd ed.* (DDIA); the design generalizes to
other sources later.

It is also a deliberate engineering exercise: a persistent, testable application with a CLI, a
database, model integration behind a replaceable interface, failure handling, and evaluation.

## The three jobs (confirmed)

1. **Starting** — know exactly where to resume and what you were learning, in one command.
2. **Understanding** — get explanations, examples, and questions about difficult concepts,
   grounded in the book and your own notes.
3. **Remembering** — revisit important ideas, review on two streams (cumulative + per-chapter),
   and find gaps in understanding.

## Design principles (confirmed, carried from the HLD)

- **Local-first (for now).** Reading progress and notes live on your machine. A later hosted
  deployment is planned and must not require a rewrite — see
  [future-hosted-migration](../architecture/future-hosted-migration.md).
- **Model-independent.** Claude, OpenAI, or a local model, all behind one small adapter interface.
- **Evidence-based.** Your notes and book excerpts are kept distinct from AI-generated text, and
  explanations cite the page/section they draw from.
- **User-controlled.** You decide what to read, save, and review; your edits override generated values.
- **Small core, optional intelligence.** The non-AI workflow works fully offline. The model is an
  add-on, never a dependency of saving or resuming.

## What changed from the original HLD

The HLD was built around a *text-free* physical-book workflow (no OCR, no digital copy). During
discovery we reversed that: the reader owns and has scanned the book, so the system now **ingests
the full text** to auto-surface concepts and produce page-cited explanations. We also adopted the
HLD appendix's **multi-source** direction as the baseline data model. Both changes and their
rationale are in the [decision log](../decisions/decision-log.md) (D1, D2).

## Explicitly out of scope for the MVP

Autonomous/multi-agent orchestration, a web dashboard, and vector/embedding retrieval (deferred).
OCR is *in* scope only because the source is a scan, but figure/equation understanding and scanned-
image cleanup beyond basic text OCR are stretch. See [scope-and-roadmap](scope-and-roadmap.md).

## The one-sentence first milestone

After reading five pages of DDIA, you can save your position and one question, close the terminal,
and resume tomorrow with useful context — offline, with zero AI calls required.
