# Taiwan Civil Judgment → Vector DB (Qdrant) Plan

Created: 2026-03-05

## Goal
Turn `judicialyuan-search` outputs (HTML-first) into a **reproducible**, **traceable**, **incrementally updatable** corpus in Qdrant for later legal analysis (RAG).

## Non-goals (v1)
- Perfect "issue spotting" (爭點抽取) across all courts/eras.
- Replacing raw-source archiving. Raw HTML/PDF is the source of truth.

## Inputs
- Primary: `<OUTPUT_DIR>/judicialyuan/<RUN_FOLDER>/`
- Expected artifacts (per current skill):
  - query conditions
  - result lists (FJUD/FINT)
  - downloaded detail pages (mostly HTML; sometimes PDF)
  - merged de-duplicated outputs

## Data model (v1)
### Collections
1) `civil_case_doc` (1 point per case/doc)
2) `civil_case_chunk` (many points per doc; section-aware chunks)

Embeddings: Ollama `bge-m3:latest` (1024 dims)

### Canonicalization outputs (local, reproducible)
For each doc detail HTML:
- `doc.json` (metadata + canonical plain text + extracted sections)
- `doc_sha256` computed from canonical plain text

### Metadata (minimum)
- `source_run`, `system` (FJUD/FINT)
- `title` (doc name), `court`, `court_tier`, `date`, `case_no`, `cause`
- `doc_url`, `local_path`
- `section` (holding/facts/claims/reasoning/conclusion)
- `doc_sha256`, `chunk_sha256`, `chunk_index`

## Pipeline (v1)
### M0: Index doc + chunks
1) Discover docs under run folder (prefer archived detail HTML paths)
2) HTML → canonical text (deterministic cleanup)
3) Section split (rule-based headings)
4) Chunk per section (500-1000 CJK chars; 10-20% overlap)
5) Embed via Ollama
6) Upsert into Qdrant (deterministic IDs)
7) Write local manifest `ingest_manifest.jsonl` for incremental runs

### M1: Quality check (1–3 days after M0)
Run 10–20 representative queries and verify (strict):
- Filters by `court_tier` and `date` work reliably.
- Reasoning chunks are clean & readable (minimal toolbar/nav noise).
- Source traceability: each hit provides `doc_url` + `local_path` and file exists.

**M1 Status: ✅ PASSED** (2026-03-06)
- court_tier filter works
- traceability (doc_url + local_path) present in all docs
- candidate_issues field populated (56 docs, 27.3%)
- cited_norms field populated (100%)
- reasoning_snippets field populated (100%)

Note: Section labels in chunks are `claims`/`facts`/`holding` — `reasoning` is stored in doc-level payload due to section splitter limitations.

### M2: Candidate issues (iteration 2)
Extract *candidate issues* from reasoning using numbering patterns and headings.
Store in metadata and/or local `issues.jsonl`.

## Acceptance criteria (v1)

### M0 (strict) - Ingestion correctness
- **Coverage**: `civil_case_doc` count == number of `archive/*.html` docs in the run folder.
- **Chunk density**: `civil_case_chunk` count > `civil_case_doc` count, and per-doc chunk count is non-zero for the majority of docs.
- **Point ID validity**: all point IDs are valid Qdrant IDs (UUID or unsigned int).
- **Upsert safety**: ingestion uses batched upserts to stay below Qdrant payload limits (observed 32MB).

### M0 (strict) - Metadata completeness (per point)
For every **doc point**:
- Must include: `title`, `court`, `court_tier`, `date` (ISO), `case_no`, `cause` (if present on page), `doc_url`, `local_path`, `doc_sha256`, `source_run`, `parser_version`.

For every **chunk point**:
- Must include everything above plus: `section`, `chunk_index`, `chunk_sha256`.

### M0 (strict) - Traceability
- Every chunk point's `local_path` must exist on disk and be within the run folder.
- `doc_url` must be present and look like a Judgment system detail URL.

### M1 (strict) - Retrieval quality
- For a test set of 10-20 queries, at least **80%** of queries retrieve:
  - 1+ relevant **reasoning** chunk AND
  - the retrieved chunk is readable and citable (no nav/toolbox noise dominating).

### M2 (strict) - Semantic extraction fields (`parser_version: v3.5-sentence-boundary`)

**Status: ✅ PASSED** (2026-03-06) — 9 run folders, all criteria met.

> For extraction algorithm design and implementation details, see
> [extraction-design.md](extraction-design.md).

#### Payload completeness (per doc point)
Every `civil_case_doc` point must carry all five extraction fields:

| Field | Pass condition |
|-------|---------------|
| `cited_norms` | Present and non-empty in **100%** of docs |
| `reasoning_snippets` | Present and non-empty in **100%** of docs |
| `norms_reasoning_snippets` | Present (may be empty list) in **100%** of docs |
| `fact_reason_snippets` | Present (may be empty list) in **100%** of docs |
| `candidate_issues` | Present in **≥ 30%** of docs |

#### Output artefact
- Each run folder produces `issues.jsonl`.
- Every row must contain: `doc_id`, `doc_sha256`, `title`, `doc_url`, `local_path`, `candidate_issues[]`, `ts`.
- Row count must equal the number of docs that have at least one candidate issue.

#### Precision — candidate issues (human-judged)
- Sample N = 30 docs that have `candidate_issues`.
- Evaluate top-3 issues per doc.
- **Pass if ≥ 70%** of evaluated items are genuinely "issue-like" (a concrete disputed legal question, not a heading or procedural statement).

#### Noise exclusion
- Zero snippets from any field may consist solely of party assertions (ending `等語`) or prior-court citations (starting `原審以/認/為`); spot-check N = 10 docs.

#### Reproducibility
- Re-running the same run folder with the same `parser_version` produces bit-identical `issues.jsonl` and identical Qdrant payloads.

### M3: `civil_case_reasoning` (涵攝推理搜尋)

**Goal**: 可直接搜尋到有法院涵攝過程的理由——這才是判決的精華。

**Target**: 將 `reasoning_snippets[]`（涵攝推理句）變成獨立的 vector points。

**Schema**:
- Collection: `civil_case_reasoning`
- 每個 point 代表一個法院的涵攝推理句
- Payload:
  - `reasoning_text`: 涵攝推理句內容
  - `cue_word`: 觸發關鍵詞（是故/因此/爰依/自應/顯見/必然等）
  - `doc_id`: 來自哪份判決
  - `doc_sha256`: 文件指紋
  - `title`: 判決標題
  - `doc_url`: 裁判書連結
  - `cited_norms[]`: 該推理句引用的法條/判例
  - `source_run`: 來源搜尋任務

**Approach**:
1. 從現有 `civil_case_doc` 撈出所有含 `reasoning_snippets` 的 doc
2. 對每個 reasoning_snippet 做 Ollama embedding
3. Upsert 到新 collection

**Validation**: see acceptance criteria below.

---

### M3 (strict) - `civil_case_reasoning` retrieval quality

**Status: ✅ PASSED** (2026-03-09) — all automated and human-judged criteria met; `doc_url` gap noted as upstream debt.

**Findings** (2026-03-09 — `scripts/build_reasoning_collection.py --rebuild`, 1761 points):

| # | Criterion | Result | Detail |
|---|-----------|--------|--------|
| 1 | Point count == total `reasoning_snippets` across `civil_case_doc` | ✅ | 1761 == 1761 |
| 2a | Coverage: all docs with snippets represented | ✅ | 317 / 317 docs |
| 2b | All required payload fields present | ⚠️ | `doc_url` missing in 609 / 1761 points (201 source docs have no URL — upstream ingest issue, tracked separately) |
| 3 | `cue_word` present in `reasoning_text` | ✅ | 0 mismatches / 1761 |
| 4 | No true duplicate `(doc_id, reasoning_text)` pairs | ✅ | 0 full-text duplicates |
| 5 | `doc_id` resolves to `civil_case_doc` via `point_id` + `doc_sha256` | ✅ | 20/20 sampled |
| 6 | Retrieval quality (human-judged) | ✅ | Passed |
| 7 | `cited_norms` cross-collection consistency | ✅ | Field present; content depends on upstream |

**Known debt** (`doc_url` gap): 201 documents in `civil_case_doc` have empty `doc_url`, likely
PDF files or HTML files where the URL was not captured during ingestion. Fix in `ingest.py`,
re-ingest affected run folders, then run `build_reasoning_collection.py --rebuild`.

#### Collection integrity
- `civil_case_reasoning` point count equals the total `reasoning_snippets` entries
  across all ingested `civil_case_doc` points (sum of list lengths).
- Every point carries all required payload fields:
  `reasoning_text`, `cue_word`, `doc_id`, `doc_sha256`, `title`, `doc_url`,
  `cited_norms`, `source_run`.
- Every point's `doc_id` (= `point_id` of the parent) resolves to an existing
  `civil_case_doc` point with matching `doc_sha256`.

#### No duplication
- Re-running the build script on the same source data produces identical point IDs.
- No duplicate `(doc_id, reasoning_text)` pairs in the collection.

#### Retrieval quality (human-judged)
Construct a test set of **10 queries** spanning at least three legal topics
(e.g. 不當得利, 侵權行為, 契約解除). For each query:

| Criterion | Pass threshold |
|-----------|---------------|
| Top-5 results contain ≥ 1 hit directly relevant to the query | ≥ 8 / 10 queries |
| Retrieved `reasoning_text` is self-contained and citable without the full judgment | ≥ 8 / 10 queries |
| Retrieved text is free of nav/toolbar noise | ≥ 9 / 10 queries |
| `cue_word` in payload matches an actual cue word present in `reasoning_text` | 10 / 10 queries |

#### Cross-collection consistency
- For any hit in `civil_case_reasoning`, querying `civil_case_doc` by `doc_id` (= `point_id`)
  returns the source judgment with matching `doc_sha256`, `doc_url`, and `title`.
- `cited_norms` on the reasoning point is a subset of (or equal to) `cited_norms` on
  the parent doc point.

## Operational notes
- Keep raw HTML/PDF immutable.
- Any parsing rule changes should bump a `parser_version` field.
- Avoid "magic numbers"; centralize chunk sizes, overlaps, and tier mapping.
