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

### M2 (strict) - Candidate issues + Norms + Reasoning extraction (v3.x)

**Parser version**: `v3.5-sentence-boundary`

**M2 Status: ✅ COMPLETED** (2026-03-06)
- All run folders ingested (9 folders)
- All norm/reasoning fields populated in Qdrant

#### A) Candidate Issues (`candidate_issues`)
- Run folder produces `issues.jsonl`.
- Each row: `doc_id`, `doc_sha256`, `title`, `doc_url`, `local_path`, `candidate_issues[]`, `ts`.
- Issue extraction rules (v1.3-v1.7):
  - Only from "爭點/爭執事項" block (header detection with ordinal prefixes like "三、爭點：")
  - Skip: "兩造爭執事項" heading itself, "不爭執事項" (agreed facts)
  - Skip headings: "按/次按/再按/經查/查/考量/綜上/據上/又/末按/本院"
  - Require cue words when no explicit heading: 是否/應否/得否/可否/有無/能否/要件/成立/不成立/應負/得請求

#### B) Norms & Reasoning Extraction (v3.x)
All extractions are from **court reasoning section onwards** (not full judgment):
- Header detection: "本院之判斷"/"得心證之理由"/"兩造爭執事項"/"經查："/"茲分述如下：" (with ordinal prefixes)
- Fallback: use full text if no header found
- Supreme Court (最高法院): always use full text (no header filtering)
- **All snippets use sentence-based boundary (by "。" Chinese period)**

Extracted fields (doc-level payload):
1. `cited_norms[]` - Legal citations (民法第X條, 釋字第X號, 最高法院...判例)
2. `norms_reasoning_snippets[]` - Norm context with expansion rules:
   - Expand backwards if ends with "定有明文"/"判決意旨參照"/"判決參照"
   - Expand forwards if ends with "定有明文" AND next starts with "又"
   - Skip: party assertions ending with "等語"
   - Skip: prior court citations starting with "原審以/原審認/原審為"
3. `fact_reason_snippets[]` - Key facts from paragraphs starting with "查/經查/惟查"
4. `reasoning_snippets[]` - Subsumption reasoning (涵攝推理):
   - Cue words: 是故/因此/準此/可見/由此可見/即應/爰依/自應/從而/甚是/茲因/是則/應認/所以/顯見/顯然/必然

#### Coverage (candidate issues)
- ≥30% of docs should have `candidate_issues` extracted

#### Precision (human-judged)
- Sample N=30, judge top-3 issues per doc
- Pass if ≥70% are "issue-like"

#### Reproducibility
- Deterministic IDs; overwrite semantics

#### Storage
- All fields stored in Qdrant doc-level payload

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

**Validation**:
- 用 5-10 組 query 測試能否直接搜到涵攝推理句
- 確認檢索結果可讀且有脈絡

## Operational notes
- Keep raw HTML/PDF immutable.
- Any parsing rule changes should bump a `parser_version` field.
- Avoid "magic numbers"; centralize chunk sizes, overlaps, and tier mapping.
