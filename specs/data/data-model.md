# Data Model

SQLite for the MVP, behind a repository interface so the backend can change later (D11). The schema
generalizes the HLD's single-book tables to the multi-source model (D1), and adds the tables needed
for text ingestion (D2), concepts (D6), citations (D8), and cost logging (D10).

Conventions: primary keys are application-generated UUID/ULID strings (D11 §3). Timestamps are UTC
ISO-8601 strings. "Locator" columns store both a raw value and a display form (see
[source-abstraction](../architecture/source-abstraction.md)).

## Tables

### `sources`
Generalizes the HLD `books` table.
| field | notes |
|---|---|
| id | PK |
| title | e.g. "Designing Data-Intensive Applications" |
| source_type | enum: `physical_book` \| `markdown` \| (later) `pdf_file` \| `url` |
| uri | file path / URL; empty for a pure physical book without a file |
| edition | e.g. "2nd" |
| locator_scheme | how to parse/compare/display locators for this source |
| metadata | JSON: author, publisher, scan info, page-offset, etc. |
| created_at | |

### `sections`
Heading hierarchy (chapter → section → subsection).
| field | notes |
|---|---|
| id | PK |
| source_id | FK → sources |
| parent_id | FK → sections (nullable; null = top level) |
| title | e.g. "5.1 Leaders and Followers" |
| order_index | sort within parent |
| start_locator / end_locator | printed-page range this section spans |

### `content_chunks`
Extracted text, the citation/retrieval unit (new; enables D2/D8).
| field | notes |
|---|---|
| id | PK |
| source_id | FK → sources |
| section_id | FK → sections (nullable) |
| text | OCR/parsed text |
| locator | printed page (or heading) this chunk came from |
| order_index | reading order within the source |
| char_count / token_estimate | for budgeting prompts |

### `reading_sessions`
| field | notes |
|---|---|
| id | PK |
| source_id | FK → sources |
| started_at / ended_at | ended_at null = in progress |
| start_locator / end_locator | page range read this session |
| recap | generated recap text (nullable — session may end without one, HLD line 97) |
| recap_model_call_id | FK → model_calls (nullable) |

### `concepts`
Topics being learned. Extracted lazily up to current page, user-overridable (D6).
| field | notes |
|---|---|
| id | PK |
| source_id | FK → sources |
| name | e.g. "replication lag" |
| section_id | FK → sections (where it's introduced; nullable) |
| first_locator | earliest page it appears — used to gate surfacing (no spoilers) |
| understanding_level | enum/int: unseen → seen → shaky → solid → mastered |
| origin | enum: `generated` \| `user` \| `user_edited` (user/user_edited win over regeneration) |
| last_reviewed_at | nullable |
| created_at / updated_at | |

### `notes`
Keeps user notes distinct from AI output (HLD principle).
| field | notes |
|---|---|
| id | PK |
| session_id | FK → reading_sessions (nullable) |
| concept_id | FK → concepts (nullable) |
| content | text |
| note_type | enum: `takeaway` \| `question` \| `excerpt` \| `explanation` |
| author | enum: `user` \| `ai` |
| resolved | bool — for questions; drives "unresolved questions" surfacing |
| source_locator | page/section the note refers to (nullable) |
| model_call_id | FK → model_calls (set when author = ai) |
| created_at | |

### `citations`
Links AI-authored notes to the chunks that ground them (enforces D8).
| field | notes |
|---|---|
| id | PK |
| note_id | FK → notes (the explanation/recap/quiz item) |
| chunk_id | FK → content_chunks |
| locator | denormalized printed page for display |

### `reviews`
Recall/review history, feeds the two streams (D7) and retention metrics.
| field | notes |
|---|---|
| id | PK |
| concept_id | FK → concepts |
| stream | enum: `cumulative` \| `chapter` |
| question | text (may be generated) |
| answer | user's answer (nullable if self-graded by recall only) |
| result | enum: `correct` \| `partial` \| `incorrect` |
| reviewed_at | |
| model_call_id | FK → model_calls (nullable; set if model-generated/graded) |

### `model_calls`
Audit + cost/latency log for every AI call (D10).
| field | notes |
|---|---|
| id | PK |
| task | enum: explain \| recap \| quiz \| extract_concepts \| eval_grounding |
| provider / model_id | |
| context_ref | JSON: ids of chunks + notes sent (full text at debug level only) |
| input_tokens / output_tokens | |
| cost_estimate | |
| latency_ms | |
| ok / error_type | |
| created_at | |

### `progress` (or a view over sessions)
Current position per source: current_locator, current_section_id, last_session_id, updated_at.
Can be derived from the latest session, but stored explicitly so `resume`/`save` are a single fast
write/read with no AI and no computation (supports the HLD "fast and dependable, even offline" rule).

## Relationships (summary)

```
sources 1─┬─* sections ───* content_chunks
          ├─* reading_sessions ─* notes ─* citations ─* content_chunks
          ├─* concepts ─* reviews
          └─1 progress
notes ─? concepts        model_calls ←? notes / reviews / sessions
```

## Design details carried from the HLD (still required)

- Store chapter/section alongside page numbers (you may revisit a section). ✅ `sections` + locators.
- Keep user notes separate from AI explanations. ✅ `notes.author` + `note_type`.
- Store timestamps and the model used for generated content. ✅ `model_calls` + FKs.
- A session may end without a recap if the model call fails. ✅ `recap` nullable.
- Keep review answers and outcomes to evaluate whether the tool helps retention. ✅ `reviews`.

## Deliberately NOT in the MVP

No embeddings/vector columns yet (D5) — but `content_chunks` is the natural place to add an
`embedding` column later with zero structural change. No per-user tables (single user, D12); the
hosted/multi-user case adds a `users` table and user_id FKs (OQ6).
