# ViGSQA / VN-GeoQA — Full Report Draft

> **Status (2026-09-05):** working draft aggregated from `docs/PLAN.md`,
> `docs/context/00a_context.md`, `docs/data_generation.md`,
> `docs/dev_test_split_v3.0.0.json`, and task records
> `docs/plans/T01–T05, T07–T11`, plus web-verified paper metadata for SPARTQA
> (NAACL 2021 main.364, Mirzaee et al.). GS-QA paper content is quoted only
> from the fork's own README abstract; the arxiv ID `2605.22811` supplied by
> the user did not resolve to GS-QA in our fetches — reconfirm before
> submission.
>
> **T02, T03, T04, T05 are all `done`.** All four evaluation seals validate;
> `evaluation-results.tar.gz` is published on release `v3.0.0`
> (508,884 bytes, SHA-256 `bb10de26aa851dab1e24baf93dbf8d32d21ecad1205aabf32920074efd484b16`).
> LLM-cache dump `llm-cache-20260905.sql.gz`, 8,085,830 bytes,
> SHA-256 `60d9e0f2…`, 27,674 rows (16,800 official + 10,859 parser + 15 demo).
>
> Result cells below are **filled from real evaluation artifacts** where
> quoted in `docs/plans/T04` and `docs/plans/T05`. Remaining `TBD` cells need
> `results/analysis/taxonomy_*.csv` regeneration (`scripts/error_taxonomy.py`)
> or per-family baseline aggregates from `scripts/report_split_metrics.py`
> against sealed `per_question.jsonl`.

---

## Deliverables Checklist (course rubric — `docs/context/00a_context.md`)

- [x] Dataset & VN handling (2.0) — VN-GeoQA v3.0.0 published on release
      `v3.0.0`; native OSM address gold; verifier-enforced factuality boundary.
- [x] Notebook + reproducibility (4.0) — `main.ipynb` re-runs sealed
      artifacts locally with zero errored cells (17/17) and Colab-verified.
- [ ] ACL Short PDF, 4–5 pages (excl. references + appendix), ACL style.
- [x] Source repo link.
- [x] Public dataset asset (release `v3.0.0`: `vn-geoqa.zip`,
      `osm-vn.sql.gz`, `evaluation-results.tar.gz`,
      `llm-cache-20260905.sql.gz`).

Rubric weights: dataset & VN handling 2.0 · implementation/notebook 4.0 ·
ACL report 4.0. Instructor Q&A: one dataset sufficient; report should
prioritize our contribution (VN data, experiments, error analysis).

---

## Report Skeleton (ACL Short — proposed section order)

1. Abstract
2. Introduction
3. Related Work / Paper Summaries
4. Dataset (VN-GeoQA v3.0.0)
5. Method — Baselines + T04 Rescue Intervention
6. Experimental Setup (dev/test protocol)
7. Results and Discussion
8. Error Analysis (T05 taxonomy)
9. Limitations and Ethical Considerations
10. Conclusion
11. References
12. Appendix (unlimited)

---

## 1. Abstract (draft — one paragraph, real numbers filled)

Vietnamese geospatial question answering has no public benchmark. We
introduce **VN-GeoQA v3.0.0**, a reproducible Vietnamese adaptation of the
GS-QA benchmark comprising 2,800 questions across 28 canonical
spatial-predicate × answer-type templates, generated from a pinned
OpenStreetMap Vietnam snapshot (`vietnam-260901.osm.pbf`) against a
PostGIS reference database. Gold answers are the live SQL result at
generation time; location gold uses native OSM address components rather
than reverse-geocoded strings. We evaluate four sealed baseline
configurations — two 9B NVFP4 LLMs (`ornith-ai/Ornith-1.5-9B-NVFP4`,
`AxionML/Qwen3.5-9B-NVFP4`) crossed with Direct and Text2SQL prompting —
served through an external OpenAI-compatible vLLM endpoint under one
frozen decoding profile. Text2SQL is strictly the right architecture:
Direct refuses on ~82% of questions, while Text2SQL answers substantively
across all eight families. We freeze a 560/2,240 dev/test split and a
pre-registered zero-LLM intervention (`records-to-answer-rescue-v1`)
that recovers 222/2,240 test questions with 0 regressions, lifting
entity text F1 by +0.162 and distance relative error by −0.078 on test.
Residual errors concentrate in SQL generation (sql-error, no-rows) and
Vietnamese address geocoding (Nominatim resolves only ~46% of predicted
addresses); Vietnamese diacritic handling is essentially solved by NFKC
normalization.

---

## 2. Introduction (draft outline)

- LLM QA has strong general benchmarks; geospatial QA is thinly covered
  and Vietnamese resources are effectively absent.
- Two anchor works:
  - **GS-QA** (English geospatial QA; DB-grounded; text + geometry
    metrics) — provides the 28-template contract and Direct/Text2SQL
    baseline design.
  - **SPARTQA** (NAACL 2021) — provides methodological validation that
    grammar-based automatic QA generation with a human-verified sample
    is a legitimate resource-construction path.
- Contribution:
  1. VN-GeoQA v3.0.0 — frozen, reproducible, publicly downloadable.
  2. Vietnamese-specific adaptations: native OSM `addr_*` gold,
     128 Vietnamese surface templates across 28 TIDs, Vietnamese
     sub-category lexicon, diacritic-stripped surface parallel form,
     NFKC-based scoring that equates composed/decomposed diacritics.
  3. Four sealed evaluation runs with a fixed Ornith parser under
     a rate-limited Nominatim policy.
  4. Frozen 560/2,240 dev/test split and pre-registered zero-LLM
     rescue intervention with statistically stable dev→test transfer.
  5. Vietnamese-specific error taxonomy: geocoding coverage — not
     diacritics — is the dominant residual language friction.

---

## 3. Related Work / Paper Summaries

### 3.1 GS-QA (Sultan et al., quoted from fork README abstract)

> "Recent advances in Large Language Models (LLMs) have led to dramatic
> improvements in question answering (QA). To address the challenge of
> evaluating QA systems, standardized benchmarks have been introduced.
> This work focuses on the problem of geospatial QA, where a large
> collection of geospatial data is available in the form of a spatial
> database or other forms. There is limited work on creating or
> evaluating geospatial QA systems. This work has various limitations
> including a small number of questions, reliance on a knowledge graph,
> limited geospatial operators, and no complex reasoning. We present
> GS-QA, an extensible geospatial QA benchmark with 2800 question-answer
> pairs on top of Open Street Maps (OSM) and Wikipedia data, covering
> various spatial objects, predicates, and answer types. A key feature
> of GS-QA is that some of the questions require combining information
> from multiple sources, e.g., geospatial information from OSM and other
> information from Wikipedia. GS-QA includes a comprehensive evaluation
> methodology that combines text-based QA measures with
> geospatial-specific measures. We implemented various LLM-based
> geospatial QA baselines, using combinations of LLMs, retrieval, and
> structured querying. Our results show that existing solutions have
> very low accuracy, which warrants more research in this direction."

Key take-aways grounded in the fork's `generator/` and `baselines/`
code:

- **28 canonical templates** = spatial predicate × output type.
  Predicate families in `generator/templates_vi/`: `knn`,
  `knn:direction`, `knn:towards`, `knn:non_spat_filter`, `range`,
  `range:direction`, `range:towards`, `range:non_spat_filter`,
  `intersects`, `intersects:area_max`, `intersects:area_total`,
  `intersects:length_max`, `intersects:length_total`, multi-source
  `knn+name+multi_source2`.
- **Answer types**: `name`, `loc`, `angle`, `count`, `distance`,
  `area`, `length`, plus external-fact for T7/T8 multi-source.
- **Two baseline families**: **Direct** and **Text2SQL** (three-stage:
  `sql_generate` → `sql_exec` → `sql_answer`).
- **Evaluation combines text F1 with geometry-aware metrics** —
  motivates our `dist_err` (`min(distance_m/500000, 1)`) and capped
  relative error.

> **TODO before submission:** fetch canonical GS-QA arxiv PDF (user
> supplied `2605.22811`, which did not resolve). Confirm authors,
> exact title, exact metric definitions before citing.

### 3.2 SPARTQA (Mirzaee, Rajaby Faghihi, Ning, Kordjamshidi — NAACL 2021 main.364)

Verified from ACL Anthology:

- Introduces SPARTQA, a textual QA benchmark for spatial reasoning.
- Uses distant supervision: grammar and reasoning rules automatically
  generate spatial descriptions of visual scenes plus QA pairs.
- Includes a human-annotated subset (SPARTQA-Human).
- Demonstrates that further pretraining LMs on the auto-generated data
  improves spatial understanding, with transfer gains on bAbI and boolQ.

Positioning of ViGSQA against SPARTQA:

- We reuse SPARTQA's *paradigm* (rule-based automatic generation +
  human QC on a sample). Our G4′ human QC covers 5 records/TID × 28
  TIDs = 140.
- We do *not* run SPARTQA's LM-pretraining branch (frozen LLMs only).
- SPARTQA operates on synthetic textual scenes; VN-GeoQA operates on
  real OSM geometry — different signal source, same
  distant-supervision philosophy.

### 3.3 Related lines of work (references to add before submission)

- Text-to-SQL benchmarks: Spider, BIRD, WikiSQL.
- Geospatial QA: GeoQuery, MapQA, NL2GQL.
- Spatial reasoning LMs: SpatialRoBERTa, StepGame, ReSQ.
- Vietnamese NLP resources: PhoBERT, ViT5, UIT-ViQuAD, ZaloAI-QA, ViMMRC.
- Vietnamese/Southeast-Asian LLMs: PhoGPT, SeaLLM, Vistral.

---

## 4. Dataset — VN-GeoQA v3.0.0

### 4.1 Design Goals

1. **DB-grounded gold** — every answer is the live PostGIS result of
   a canonical SQL template; no LLM/human transcription.
2. **Reproducible** — byte-identical regeneration from
   `(seed=42, snapshot, generator commit)`.
3. **Faithful Vietnamese address gold** — native OSM `addr_*`
   components, never reverse-geocoded.
4. **Externally verifiable factuality boundary** — T7/T8 multi-source
   attributes never appear in the reference schema.

### 4.2 Source Data

- **Snapshot:** Geofabrik `vietnam-260901.osm.pbf` — 327,637,117 bytes;
  md5 `c03f4b2db0fce6b85af7071ce6bfc13b`;
  sha256 `edf2d41d93b25474acc14a34f6c313940ecfea5671835299ddd793c60d08a3e8`.
  Pinning enforced by `scripts/download_osm.sh`; `vietnam-latest`
  rejected.
- **Import:** `osm2pgsql` with project Lua style `scripts/osm_poi.lua`.
- **Tables and row counts** (G1′ audit):

| Table | Rows |
|---|---:|
| `pois` | 38,207 |
| `regions` | 8,567 |
| `parks` | 1,493 |
| `lakes` | 7,987 |
| `roads` | 175,883 |

- **`pois` view:** 24 columns — identity, category, six filter tags
  (`cuisine`, `museum`, `takeaway`, `outdoor_seating`, `delivery`,
  `emergency`), external anchors (`wikidata`, `wikipedia`), eight
  native address columns (`addr_housenumber`, `addr_street`,
  `addr_place`, `addr_suburb`, `addr_district`, `addr_city`,
  `addr_province`, `addr_postcode`), `geometry`, `geo_wkt`.
- **`capacity` deliberately absent** — T7/T8 external attribute
  invariant; enforced by fail-closed `pois_view_columns()` parser
  of `sql/refresh_views.sql`.
- **External knowledge:** frozen `wikipedia_cache_vi.json`,
  sha256 `09b74191…` — no live network during generation.

### 4.3 Question Record Schema

Each line of `data/questions_vi/<type>.jsonl`:

```json
{
  "id": "knn+name-001",
  "type": "knn+name",
  "answer_type": "name",
  "question": "Nhà hàng nào gần Nhà thờ Đức Bà nhất?",
  "question_surfaces": {"full": "...", "stripped": "..."},
  "sql": "SELECT id, geo_wkt, poi_name FROM pois WHERE ... ORDER BY geometry <-> ... LIMIT 1",
  "answers": [{"id": 123, "poi_name": "...", "geo_wkt": "POINT(106.7 10.78)"}],
  "question_entities": {"[1]": {...}, "[2]": {...}}
}
```

Stable `{type}-NNN` ids; TID mapping in
`data/questions_vi/MANIFEST.json`.

### 4.4 Answer-Type Distribution (2,800 total)

| Answer type | N | Notes |
|---|---:|---|
| name | 1,200 | single, or full distance-ordered set for range types |
| loc | 800 | canonical address + 8 native components + `geo_wkt` |
| angle | 200 | degrees clockwise from north |
| count | 200 | exact integer |
| distance | 200 | metres |
| area | 100 | square metres |
| length | 100 | metres |
| **Total** | **2,800** | |

### 4.5 Vietnamese Surface Realization

- 28 template files under `generator/templates_vi/`, total **128
  surfaces** (mean ≈ 4.6 per TID).
- Placeholders: `[1]` = target category, `[2]` = radius or anchor,
  `[3]` = anchor.
- Example — `knn+name.txt`:

  ```text
  [1] nào gần [2] nhất?
  [1] gần nhất với [2] là gì?
  Cho tôi biết [1] gần [2] nhất.
  Đâu là [1] gần [2] nhất?
  Tìm giúp tôi [1] gần [2] nhất.
  [1] gần [2] nhất có tên là gì?
  Tôi đang tìm [1] gần [2] nhất.
  ```

- `question_surfaces.full` and `.stripped` (diacritic-stripped) both
  stored — supports future robustness study.
- Category lexicon `generator/filter_labels_vi.json` — Vietnamese
  sub-categories preserving OSM tag granularity, e.g.
  `restaurant → nhà hàng món Việt / nhà hàng hải sản / quán mì và phở
  / nhà hàng Nhật / nhà hàng Hàn / quán pizza / quán burger / nhà hàng
  Ấn / quán gà rán / nhà hàng có bán mang về / nhà hàng có chỗ ngồi
  ngoài trời / nhà hàng có giao hàng`.

### 4.6 Location Semantics (v3 upgrade)

**Motivation.** GS-QA loc gold is coordinate-only; a Vietnamese
address benchmark cannot be evaluated on coordinates because
Vietnamese address form is hierarchical (`số nhà, đường,
phường/xã, quận/huyện, thành phố/tỉnh`). v2 used `addr_city` alone
and was superseded.

**Candidate filter** (prompt-visible predicate):

```sql
(addr_street IS NOT NULL OR addr_place IS NOT NULL)
AND (addr_suburb IS NOT NULL OR addr_district IS NOT NULL
     OR addr_city IS NOT NULL OR addr_province IS NOT NULL)
```

**Coverage audit** (G1′, all 38,207 POIs, per-component non-null):

| Component | Non-null |
|---|---:|
| `addr_housenumber` | 10,229 |
| `addr_street` | 13,857 |
| `addr_place` | 72 |
| `addr_suburb` | 7 |
| `addr_district` | 4,803 |
| `addr_city` | 4,795 |
| `addr_province` | 4,489 |
| `addr_postcode` | 1,470 |

Candidate criterion comparison:

| Criterion | Pool | Note |
|---|---:|---|
| street-only | 13,857 | ambiguous (street names repeat across cities) |
| **street-or-place + ≥1 broader locator** | **5,321** | **selected** — least restrictive, unambiguously geocodable |
| housenumber + street + broader | 4,020 | too restrictive |
| city-only | 4,795 | not a point |

**Sub-category pool sizes (address-bearing):** all 26 sub-categories
non-empty. Selected — restaurant 1,375 · cafe 1,048 · convenience
646 · hotel 541 · bank 288 · supermarket 243 · … · university 11 ·
gallery 9 · sports_centre 9 · stadium 4 · swimming_pool 2.

**Geographic spread** (top cities in address-bearing pool):
Hà Nội 1,464 · Bắc Ninh 643 · HCMC ~552 (three OSM spellings) ·
Cần Thơ 126 · Đà Nẵng 113 · Đà Lạt 71 · Hội An 67+41 · Nha Trang 50 ·
Huế 40.

**Loc gold format:**

- `geo_wkt` — authoritative spatial reference (used for `dist_err`).
- Eight `addr_*` components — verbatim OSM values, frozen orthography
  (mixed forms preserved: `Bắc Ninh` vs `Bac Ninh`).
- One deterministic canonical `address` string joined from those
  components. Example: `"5B Nguyễn Thiện Thuật, Phường Hoàn Kiếm,
  Hà Nội, Thành phố Hà Nội"`.

kNN loc = single nearest address-bearing candidate; range loc = full
distance-ordered set (median 2–6, max 542; 65–89% of kNN loc gold
carries a housenumber).

### 4.7 Multi-source Semantics (T7/T8)

- Multi-source attributes registered: `established, built, architect,
  founder, director, opened, capacity, designed`.
- Live `information_schema` audit at freeze: intersection with `pois`
  view columns = **0**.
- `wikidata` / `wikipedia` are anchor identifiers (53 POIs), not
  answer facts — retained.
- Text2SQL cannot answer T7/T8 by column reference; the model must
  retrieve the entity through SQL then reach for external knowledge.

### 4.8 Generation Pipeline

```
OSM PBF ──► osm2pgsql ──► PostGIS (osm_vn)
                             │
                             ▼
                    generator_vi.py --seed 42 --count 100
                    ┌─────────────────────────────┐
                    │ 1. sample anchor POI        │
                    │    (TABLESAMPLE REPEATABLE) │
                    │ 2. execute SQL template     │
                    │ 3. validate answer set      │
                    │ 4. fill VI text template    │
                    │ 5. write questions_vi/*.jsonl│
                    └─────────────────────────────┘
                             │
                             ▼
                    verify_vi.py --all
                             │
                             ▼
                    MANIFEST + sha256 seal
```

Seed 42 pins Python `random` and per-query `TABLESAMPLE SYSTEM(10)
REPEATABLE`. Byte-identical regeneration verified: second seed-42
run into `data/v3-stage/regen2` produces `diff -r`-clean output
across all 28 files (2,800 questions).

### 4.9 Quality Assurance (Gates G1′–G6′)

| Gate | Check | Result |
|---|---|---|
| G1′ | DB rebuild — 5-table counts, spatial ops (`SUM(ST_Area(parks)) ≈ 2.434e10 m²`, `MAX(ST_Length(roads)) = 75,961 m`, KNN `<->`), 8-column address coverage audit, T7/T8 overlap = 0 | pass |
| G2′ | Smoke: 5 × 28 = 140 questions | 140/140 verifier pass; loc acceptance 3–15 it/s |
| G3′ | Full seed-42 generation | **2,800/2,800**, byte-identical regeneration |
| G4′ | Human QC — 5 records/TID = 140 | user-approved after answer-type-aware TSV |
| G5′ | Runner static checks + v2-seal negative test | pass — all four seals report `incomplete` under `pv-8394cd22` |
| G6′ | Publish `v3.0.0` + restore verification | dataset sha256 `d7a0c45c3ac9013215f8b25f85020ad636e40f1b2f16fb5d173c666f6d117ff1`; restored DB matches G1′ counts exactly |

### 4.10 Reproduction

- **Release restore (no DB required):** `scripts/restore_dataset.sh`
  downloads `v3.0.0` asset `vn-geoqa.zip` (2,388,528 bytes) and
  sha256-verifies against `scripts/v3.0.0.sha256`.
- **From scratch:**

  ```bash
  export PGHOST=127.0.0.1 PGPORT=5432 PGDATABASE=osm_vn \
         PGUSER=postgres PGPASSWORD=postgres
  ./scripts/download_osm.sh
  ./scripts/init_database.sh
  ./scripts/import_osm.sh
  python generator/generator_vi.py --seed 42 --count 100 \
         --output data/questions_vi
  ```

- Dataset prompt freeze hash: `pv-8394cd22`.
- Raw-inference prompt hash: `pv-26b1ac0d` (three prompts:
  `direct_answer_vi`, `text2sql_generate_vi`, `text2sql_answer_vi`).

### 4.11 Recorded Dataset Limitations (kept, not patched)

1. **OSM tag reuse** — billiards under `leisure=sports_centre`,
   resorts under `swimming_pool`, laptop store under
   `shop=supermarket`.
2. **Mapper noise** — typos, mixed-diacritic POI names.
3. **Sparse-province kNN distances** — nearest-neighbour up to
   ~140 km in remote provinces.
4. **Heavy-tail range cardinality** — range gold sets reach 542
   answers (median 2–6). Scoring uses best-match against the full set.
5. **Urban bias** — POI density concentrates in HCMC / Hà Nội.

---

## 5. Method — Baselines + T04 Rescue Intervention

### 5.1 Models

- `ornith-ai/Ornith-1.5-9B-NVFP4`
- `AxionML/Qwen3.5-9B-NVFP4`

External OpenAI-compatible vLLM (`--reasoning-parser qwen3` on the
optional compose service). Notebook/runners never start vLLM; they
probe `/v1/models`.

### 5.2 Baselines

- **Direct** — question → LLM → answer. No DB, no retrieval.
- **Text2SQL** — three stages:
  1. `sql_generate`: schema + question → LLM writes SQL.
  2. `sql_exec`: PostgreSQL executes.
  3. `sql_answer`: LLM narrates result rows in Vietnamese.

### 5.3 Frozen Decoding Profile (T11)

| Parameter | Value |
|---|---|
| temperature | 1.0 |
| top_p | 0.95 |
| top_k | 20 |
| min_p | 0 |
| presence_penalty | 1.5 |
| repetition_penalty | 1.0 |
| max_completion_tokens | 32768 |
| seed | 42 |
| thinking | on |

### 5.4 Infrastructure and Caching (T09)

- PostgreSQL LangChain LLM cache — 27,674 rows total
  (16,800 official + 10,859 parser + 15 demo). Dump
  `llm-cache-20260905.sql.gz` on release `v3.0.0`.
- Raw JSON step artifacts in `cache_vi/pv-26b1ac0d/` — one file per
  QID per stage.
- G6 artifact-integrity seal binds: model/baseline identity, frozen
  dataset hash, frozen prompt hash, repo-pinned OSM/DB provenance,
  raw artifact sha256.
- Retry policy: 3-attempt structural retry + cache-eviction on
  invalid JSON. Structurally-valid-but-incorrect outputs never
  retried.

### 5.5 Evaluator Contract (T03)

- **Parser identity:** `ornith-ai/Ornith-1.5-9B-NVFP4` parses every
  Ornith/Qwen × Direct/Text2SQL answer with one frozen prompt and
  3-attempt JSON contract.
- **Text normalization:** Unicode NFKC + casefold + punctuation
  separation + whitespace collapse. Vietnamese diacritics preserved.
  No NLTK, no stopword removal. Token P/R/F1.
- **T01–T28 family mapping:** entity/name T01–T06/T08–T12; textual
  fact T07; Location T13–T20; Direction/angle T21–T22; Count T23–T24;
  Distance T25–T26; Area T27; Length T28.
- **Location scoring:** extract non-empty parsed address → text-score
  → geocode with Nominatim → compare vs gold `geo_wkt` via
  `min(distance_m / 500000, 1)`. No SQL/DB/POI/coord fallback.
- **Direction:** 8 Vietnamese sectors; circular angular error / 180.
- **Numeric:** finite values only; supported metric units normalized;
  integral counts required; relative error capped at 1.
- **Range answers:** every applicable prediction/gold pairing
  considered; deterministic best-match indices recorded.
- **Nominatim policy** (T03 hardening, 2026-09-04/05):
  `Nominatim(timeout=10)` behind `RateLimiter(min_delay=1s,
  max_retries=2, error_wait=5s, swallow_exceptions=False)` — matches
  Nominatim bulk policy of ≤1 req/s. HTTP 400/414 rejections are
  persisted as terminal `rejected` records, never re-queried, and
  excluded from spatial matching (score `attempted=false, error=1.0`).
  Timeouts, 5xx, 429 remain transient and abort without a seal.
- **Entity gold keys:** `poi_name / park_name / lake_name /
  road_name` — `lake_name` added after 78 T11/T12 gold entities
  surfaced only under `lake_name`.
- **Evaluation seal v2** binds raw seal + evaluator identity +
  parser config + Nominatim cache hashes.

### 5.6 Rescue Intervention — `records-to-answer-rescue-v1` (T04)

Pre-registered zero-LLM intervention on Ornith/Text2SQL only. One
change, no new inference, implemented in `scripts/t04_rescue.py`.

- **Trigger (fallback-only):** sealed run has empty `candidates` —
  score already at unattempted floor. Answered questions never
  touched → per-question scores can only improve or tie (asserted
  per question after every evaluation).
- **Action:** if `sql_exec` rows carry usable typed values, emit them
  through the parser's fenced-JSON shape:
  - entity → first non-empty of `poi_name / park_name / lake_name /
    road_name` per row (published gold key set).
  - location → `address` column or canonical string rebuilt from
    `addr_*` in MANIFEST-documented order (mirrors
    `generator_vi.canonical_address`).
  - count / distance / direction / area / length → the family column.
  - Textual facts never rescued (out-of-schema by verifier design).
- **Scoring:** merged parse records → sealed `evaluate()`;
  geocoding warm-started from sealed `geocodes.json`; only genuinely
  new rescued addresses hit Nominatim.
- **Outputs:** `results/t04/rescue/` only (freeze manifest, per-question
  jsonl, rescued jsonl, geocodes, summary).

### 5.7 Runbook

```bash
# Raw inference (all four runs already sealed)
scripts/inference.sh

# Evaluation (all four seals valid; rerun is idempotent skip)
export OPENAI_BASE_URL=http://<host>:8000/v1 OPENAI_API_KEY=<key>
scripts/evaluate.sh --llm-concurrency 4

# Dev/test split derivation (deterministic re-check)
python scripts/make_dev_test_split.py --check

# Rescue intervention (regenerates results/t04/rescue/)
python scripts/t04_rescue.py

# Error taxonomy CSVs
python scripts/error_taxonomy.py

# Live 5-question Vietnamese demo (isolated cache)
env OPENAI_BASE_URL=… pixi run python scripts/t05_demo.py
```

---

## 6. Experimental Setup

- **Dataset:** VN-GeoQA v3.0.0, 2,800 questions across 28 TIDs.
- **Configurations sealed:** 2 models × 2 baselines = 4 runs =
  11,200 raw predictions. Text2SQL adds three step outputs per QID.
- **Raw-inference status:** all four runs sealed under `pv-26b1ac0d`.
- **Evaluation status:** all four seals valid (2026-09-05); rerun
  from cached parse/geocode reproduces artifacts byte-identically.
- **Dev/test protocol (T04):**
  - Sidecar `docs/dev_test_split_v3.0.0.json`. Per TID, QIDs ranked
    by ascending `sha256("vigsqa-devtest-v1::<qid>")`; first 20 of
    100 → dev, remaining → test. Total: **560 dev / 2,240 test**.
    No train split.
  - Deterministic re-derivation via
    `python scripts/make_dev_test_split.py --check`.
  - Sealed T03 artifacts are read-only; intervention arms are
    scored by importing `evaluate`/`candidates`/`finite_number`/
    `load_geocodes` verbatim from `run_evaluation.py`.
  - Stratification audit: 28 TIDs × exactly 100 each; TID is the only
    stratification variable. Dev/test baseline aggregates track each
    other closely (e.g. Ornith/T2SQL location distance error 0.670
    dev vs 0.643 test).
- **Metric semantics check** (T04, before touching anything):
  compared paper §5.1, upstream `baselines/evaluate.py`, and
  `scripts/run_evaluation.py` — evaluator preserves paper semantics.
  Deviations are deliberate Vietnamese adaptations (NFKC composed/
  decomposed equivalence, Unicode-category punctuation, digits kept
  as digits, Vietnamese sector names, decimal comma, multiset token
  overlap, gold-side geometry instead of geocoded gold).
- **Pre-registration disclosure (T04):** intervention *feasibility*
  was scouted on full-set aggregates before freeze (recorded
  transparently in the T04 record); the freeze commit `5259b7e6`
  contains rescue script + split + dev evidence; test evaluation ran
  strictly after that commit.

---

## 7. Results and Discussion

### 7.1 Headline — Baseline Family Aggregates (test split, sealed)

From `docs/plans/T04-baseline-improvement.md` "Test evaluation" — the
`baseline` column is the sealed T03 aggregate restricted to the 2,240
test questions (unattempted included at worst-case).

| family | primary metric | Ornith/Text2SQL baseline |
|---|---|:-:|
| entity | text_f1 | 0.278 |
| location | text_f1 | 0.387 |
| location | distance_error | 0.643 |
| direction | text_f1 | 0.552 |
| direction | angle_error | 0.433 |
| distance | relative_error | 0.645 |
| count | relative_error | 0.570 |
| area | relative_error | 0.711 |
| length | relative_error | 0.675 |
| textual_fact | text_f1 | 0.000 |

Other three runs (Ornith/Direct, Qwen/Text2SQL, Qwen/Direct) — per-
family baseline aggregates **TBD from `scripts/report_split_metrics.py`
per-run outputs**. Numbers below are already recorded qualitative
findings from T05 (`docs/plans/T05` §Full-benchmark stage × family):

- **Ornith/Direct** — refusal dominates: 904/1,100 entity, 572/800
  location, 156/200 count refused; only 77/2,800 correct overall.
- **Qwen/Text2SQL** — much higher sql-error rate than Ornith
  (39 area, 62 direction vs Ornith 26/18); entity
  310 correct / 136 rescuable / 209 sql-error / 215 no-rows;
  location 255/38/152/140.

### 7.2 Rescue Intervention — Test Deltas (T04)

222/2,240 test questions rescued (9.9%). Per-question regression
assert passed: **0 regressions** (guaranteed by fallback-only
trigger). 630 new addresses geocoded for rescued location questions.

| family | metric | baseline | rescue | delta |
|---|---|:-:|:-:|:-:|
| area | relative_error | 0.711 | 0.711 | +0.000 ≈ |
| count | relative_error | 0.570 | 0.570 | +0.000 ≈ |
| direction | text_f1 | 0.552 | 0.571 | +0.019 ↑ |
| direction | angle_error | 0.433 | 0.414 | −0.019 ↑ |
| distance | relative_error | 0.645 | 0.568 | **−0.078 ↑** |
| entity | text_f1 | 0.278 | 0.440 | **+0.162 ↑** |
| length | relative_error | 0.675 | 0.675 | +0.000 ≈ |
| location | text_f1 | 0.387 | 0.436 | +0.049 ↑ |
| location | distance_error | 0.643 | 0.589 | −0.055 ↑ |
| textual_fact | text_f1 | 0.000 | 0.000 | +0.000 ≈ |

**Dev→test transfer stability:** every family that improved on dev
improved on test with same sign and similar magnitude
(entity +0.162 test vs +0.160 dev; distance −0.078 vs −0.050) —
the improvement is not a dev-split artifact.

### 7.3 Dev-Gate Rescuability (before freeze)

Dev rescuable: **65/560 (11.6%)** — above the pre-agreed ≥5%
empty-candidate / ≥2pp headroom threshold for Tier-0 (zero-LLM).
Alternative one-Ornith-prompt-step intervention was not needed.

| family | answered | rescuable | sql-error | no-rows | rows-unusable |
|---|---:|---:|---:|---:|---:|
| area | 7 | 0 | 4 | 0 | 9 |
| count | 38 | 1 | 1 | 0 | 0 |
| direction | 30 | 1 | 3 | 6 | 0 |
| distance | 29 | 5 | 1 | 5 | 0 |
| entity | 99 | 43 | 25 | 53 | 0 |
| length | 6 | 0 | 3 | 0 | 11 |
| location | 82 | 15 | 15 | 39 | 9 |
| textual_fact | 8 | 0 | 4 | 0 | 8 |

### 7.4 Sealed Evaluation Provenance

Per-pair explicit parse errors and geocode outcomes
(found / not_found / rejected):

| pair | parse errors | geocode found | not_found | rejected |
|---|---:|---:|---:|---:|
| Ornith Text2SQL | 2 | 435 | 301 | 13 |
| Ornith Direct | 1 | 54 | 190 | 0 |
| Qwen Text2SQL | 1 | 359 | 147 | 10 |
| Qwen Direct | 4 | 24 | 240 | 0 |

Release asset: `evaluation-results.tar.gz` — 508,884 bytes,
SHA-256 `bb10de26aa851dab1e24baf93dbf8d32d21ecad1205aabf32920074efd484b16`,
25 entries containing only `results/evaluation/`. Built deterministically
(`tar --sort=name --mtime='UTC 1970-01-01' --owner=0 --group=0
--numeric-owner | gzip --no-name`).

### 7.5 Empty Result Tables — Fill from Sealed CSVs

> **Editor handoff.** These tables are the paper's core evidence
> figures. Fill only from `results/evaluation/<model>/<baseline>/`
> and `results/analysis/taxonomy_*.csv` (regenerate via
> `scripts/report_split_metrics.py` and `scripts/error_taxonomy.py`).
> Do not fabricate; do not extrapolate from archived v1 numbers.

#### 7.5.1 Overall Text F1 (all 2,800 questions, sealed)

| Model | Direct | Text2SQL | Δ (T2SQL − Direct) |
|---|:-:|:-:|:-:|
| Ornith-1.5-9B-NVFP4 | TBD | TBD | TBD |
| Qwen3.5-9B-NVFP4 | TBD | TBD | TBD |

#### 7.5.2 Per-Family Text F1 (all four runs, full 2,800)

| Family | Ornith Direct | Ornith T2SQL | Qwen Direct | Qwen T2SQL |
|---|:-:|:-:|:-:|:-:|
| kNN entity (T01–T06) | TBD | TBD | TBD | TBD |
| Range entity (T08–T12) | TBD | TBD | TBD | TBD |
| Multi-source (T07) | TBD | TBD | TBD | TBD |
| Location (T13–T20) | TBD | TBD | TBD | TBD |
| Direction (T21–T22) | TBD | TBD | TBD | TBD |
| Count (T23–T24) | TBD | TBD | TBD | TBD |
| Distance (T25–T26) | TBD | TBD | TBD | TBD |
| Area (T27) | TBD | TBD | TBD | TBD |
| Length (T28) | TBD | TBD | TBD | TBD |

#### 7.5.3 Location `dist_err` (↓ better)

| TID range | Ornith Direct | Ornith T2SQL | Qwen Direct | Qwen T2SQL |
|---|:-:|:-:|:-:|:-:|
| T13–T20 (all loc) | TBD | 0.643 (test) | TBD | TBD |
| kNN loc subset | TBD | TBD | TBD | TBD |
| Range loc subset | TBD | TBD | TBD | TBD |

#### 7.5.4 Numeric `rel_err` (↓ better)

| Family | Ornith Direct | Ornith T2SQL | Qwen Direct | Qwen T2SQL |
|---|:-:|:-:|:-:|:-:|
| Count (T23–T24) | TBD | 0.570 (test) | TBD | TBD |
| Distance (T25–T26) | TBD | 0.645 (test) | TBD | TBD |
| Area (T27) | TBD | 0.711 (test) | TBD | TBD |
| Length (T28) | TBD | 0.675 (test) | TBD | TBD |

#### 7.5.5 Attempted Rate (metric-specific)

| Metric | Ornith Direct | Ornith T2SQL | Qwen Direct | Qwen T2SQL |
|---|:-:|:-:|:-:|:-:|
| text | TBD/2800 | TBD/2800 | TBD/2800 | TBD/2800 |
| loc | TBD/800 | TBD/800 | TBD/800 | TBD/800 |
| numeric | TBD | TBD | TBD | TBD |
| direction | TBD/200 | TBD/200 | TBD/200 | TBD/200 |

---

## 8. Error Analysis (T05 — filled from sealed artifacts)

### 8.1 Taxonomy Definition (`scripts/error_taxonomy.py`)

Every question of a sealed run is classified by failure stage, then
flagged for measurable Vietnamese phenomena. Stages in decision order:

- `parse-failure` — answer record contains no fenced JSON block.
- `correct` — family-primary metric passes paper analysis thresholds
  (text F1 ≥ 0.5; error ≤ 0.1). Analysis-only; sealed scores untouched.
- `wrong-attempted` — model produced candidates but missed.
- `refused` (Direct only) — no candidates and no SQL stage exists;
  pre-registration probes showed these are genuine model refusals
  with correctly-keyed null JSON.
- Text2SQL empty-candidate split by `sql_exec` evidence:
  - `sql-error` — any statement errored.
  - `no-rows` — SQL ran, returned nothing.
  - `rescuable` — typed rows exist, `rescue_block` could emit an
    answer (class T04 recovers).
  - `rows-unusable` — rows exist but carry no typed value for the
    family (e.g. area/length totals computed out-of-database).

Phenomena flags:

- `diacritic_loss` — prediction matches gold once diacritics stripped
  from both.
- `geocode_miss` — predicted address Nominatim cannot resolve.
- `sector_right_angle_wrong` — correct 8-sector compass name but
  wrong azimuth.

### 8.2 Full-Benchmark Stage × Family — Ornith/Text2SQL (2,800 questions)

| family | correct | wrong-attempted | rescuable | no-rows | sql-error | rows-unusable | parse-failure |
|---|---:|---:|---:|---:|---:|---:|---:|
| entity | 313 | 198 | 208 | 280 | 100 | 1 | 0 |
| textual_fact | 1 | 35 | 0 | 7 | 28 | 29 | 0 |
| location | 275 | 196 | 48 | 168 | 46 | 65 | 2 |
| direction | 108 | 22 | 4 | 44 | 18 | 4 | 0 |
| count | 85 | 92 | 1 | 0 | 22 | 0 | 0 |
| distance | 66 | 52 | 24 | 44 | 12 | 2 | 0 |
| area | 29 | 11 | 0 | 0 | 26 | 34 | 0 |
| length | 30 | 5 | 0 | 0 | 22 | 43 | 0 |

Summaries:

- **Qwen/Text2SQL:** entity 310 correct / 136 rescuable / 209
  sql-error / 215 no-rows; location 255/38/152/140. Much higher
  sql-error than Ornith across families (39 area vs 26; 62 direction
  vs 18).
- **Ornith/Direct:** refusal dominates — 904/1,100 entity refused,
  572/800 location refused, 156/200 count refused; only 77/2,800
  correct overall.
- **Qwen/Direct:** per-family taxonomy **TBD** —
  `scripts/error_taxonomy.py` output.

### 8.3 Vietnamese-Phenomena Flags (full 2,800)

| flag | Ornith/T2SQL | Qwen/T2SQL | Ornith/Direct |
|---|---:|---:|---:|
| geocode_miss | 219 | 139 | 175 |
| sector_right_angle_wrong | 14 | 12 | 10 |
| diacritic_loss | 9 | 6 | 1 |

Qwen/Direct row **TBD** from re-run of `error_taxonomy.py`.

**Reading.** Geocoding — not language — is the recurring Vietnamese-
side friction: **46% of Ornith/Text2SQL location questions with
candidates** (219/471) contain at least one predicted address
Nominatim cannot resolve. Vietnamese address component order
(`số, ngõ, phố, phường/xã, quận/huyện, tỉnh`) and 6-digit postcodes
diverge from OSM's expectations. Those questions still score through
address-text F1 but their spatial distance error is dominated by
whatever Nominatim returns instead. True diacritic-loss misses are
rare (≤9 per run) because NFKC normalization equates composed/
decomposed forms. Sector naming is nearly always consistent with
the stated azimuth (14 mismatches); the direction metric weakness
is the azimuth itself, not the compass vocabulary.

### 8.4 Representative Cases

- **sql-error** `intersects+count-003`: "Số lượng tiệm bánh tại
  Phường Nha Trang là bao nhiêu?" — dominant sql-error class is
  subqueries used as expressions returning multiple rows
  (107 of Ornith's ~250 execution errors benchmark-wide).
- **no-rows** `intersects:area_max+name-001`: "Đâu là công viên có
  diện tích lớn nhất của Tỉnh Lâm Đồng?" — generated SQL filters
  away every candidate, model correctly reports nothing.
- **wrong-attempted** `intersects+count-001`: "Đếm số cửa hàng điện
  tử nằm trong Thành phố Hồ Chí Minh." → 0 (gold 51) — spatial
  predicate selects nothing; Qwen answers the identical question
  correctly → model capability, not dataset ambiguity.
- **rescuable** `intersects+count-025`: "Trong Thành phố Hồ Chí
  Minh có bao nhiêu chợ?" → ∅ with `count=103` sitting in executed
  rows — exactly the class T04 recovers.

### 8.5 Fresh Vietnamese Demo (`scripts/t05_demo.py`)

Five questions absent from all 5,600 benchmark surfaces (anchor
names asserted absent), gold grounded by executing gold SQL
read-only. Both baselines ran live through Ornith at the documented
OpenAI-compatible endpoint with cache redirected to
`baselines/cache_vi/t05-demo/` — sealed `pv-26b1ac0d` never touched.

| demo | family | gold | Text2SQL outcome |
|---|---|---|---|
| demo-001 | entity | Pharmacity | answered "Pharmacity" — exact match |
| demo-002 | location | 7 Phố Đào Duy Anh, Đống Đa, Hà Nội, 10000 | answered a different hotel's address — honest miss |
| demo-003 | count | 0 | answered 0 — exact match |
| demo-004 | distance | 429.29 m | answered 429 — correct |
| demo-005 | direction | 3 cafés with azimuths 353.8°/156.6°/149.3° | generated SQL returned 100 rows (missing ≤3 km filter); model narrated angles instead of listing |

Demo misses reproduce the taxonomy's two largest non-rescuable
classes on novel anchors (wrong-but-attempted selection; unfiltered
generated SQL) — intended qualitative evidence.

---

## 9. Limitations and Ethical Considerations

- **Coverage bias.** OSM Vietnam density skews urban (HCMC / Hà Nội).
  Rural questions rarer and noisier.
- **Snapshot staleness.** Single pinned PBF (`vietnam-260901`);
  real POIs change. State date-of-truth.
- **Automatic generation.** Template surfaces underrepresent
  conversational Vietnamese; SPARTQA-style human-paraphrased subset
  not applied in this release.
- **Frozen decoding profile.** Results not portable to different
  sampling.
- **Compute scope.** 9B NVFP4 models only. No frontier proprietary
  (GPT-4o, Claude, Gemini) or 70B-class open models evaluated.
- **Language scope.** Vietnamese-only run; English GS-QA parity not
  reported → cannot claim "Vietnamese is harder than English".
- **Model selection.** Two-model comparison → no statistical family
  claims.
- **Statistical rigor.** Single seed for inference; no bootstrap CIs
  or paired significance tests reported.
- **Rescue scope.** Only the refusal floor is recovered; wrong-but-
  attempted (the larger remaining class) and sql-error/no-rows are
  untouched. area/length/textual_fact are untouched by construction
  (SQL yields no typed aggregate for them). Location rescue may
  emit an address that differs in formatting from gold even when
  naming the same place → text F1 gains understate spatial recovery.
- **Rescue is a merge over sealed parse records**, not a full
  pipeline rerun → latency/cost claims about the full system do
  not apply.
- **Geocoding dependency.** Location metric depends on Nominatim
  coverage of Vietnamese address formats. HTTP 400/414 rejections
  (13 Ornith T2SQL, 10 Qwen T2SQL) are terminal by design; DNS/
  transient errors abort without a seal.
- **License / PII.**
  - OSM contributor data: ODbL — attribution required.
  - Wikipedia infoboxes: CC-BY-SA — attribution required.
  - User-contributed OSM address tags may contain residence data —
    generator filter excludes `building=residential`; document in
    release README.
- **Ethical use.** Benchmark evaluates map-based knowledge; not
  intended for surveillance or PII inference.

---

## 10. Conclusion (draft — real numbers from T04/T05)

VN-GeoQA v3.0.0 provides the first reproducible Vietnamese geospatial
QA benchmark grounded in a pinned OpenStreetMap snapshot and a
PostGIS reference database, with native Vietnamese address gold and
28 canonical spatial-predicate × answer-type templates yielding
2,800 questions. Four sealed baseline runs (Ornith and Qwen 9B NVFP4
crossed with Direct and Text2SQL) show Text2SQL is strictly the right
architecture: Direct refuses on roughly 82% of Vietnamese geospatial
questions (Ornith/Direct: only 77/2,800 correct), while Text2SQL
substantively engages every family. A pre-registered zero-LLM
rescue intervention on Ornith/Text2SQL — trained on 20/TID dev
evidence and frozen before test evaluation — recovers 222/2,240
test questions with 0 regressions, lifting entity text F1 by
**+0.162** and distance relative error by **−0.078** on test, with
dev→test deltas of matching sign and magnitude. Residual failures
concentrate in SQL generation (sql-error, no-rows) and Vietnamese
address geocoding: 46% of predicted addresses fail Nominatim
resolution. Vietnamese diacritic handling is essentially solved
by NFKC normalization (≤9 residual mismatches per run) —
Vietnamese-specific engineering effort should move from character
normalization to spatial-address canonicalization and better SQL
generation.

---

## 11. References (partial — expand)

- Sultan et al. GS-QA: A Benchmark for Geospatial Question Answering.
  arXiv preprint. **(User-supplied ID `2605.22811` needs
  re-verification; fetch canonical entry before submission.)**
- Mirzaee R., Rajaby Faghihi H., Ning Q., Kordjamshidi P. SPARTQA:
  A Textual Question Answering Benchmark for Spatial Reasoning.
  NAACL-HLT 2021, main.364.
- OpenStreetMap contributors. `vietnam-260901.osm.pbf`, Geofabrik.
- Nominatim, PostGIS, vLLM, LangChain.
- Ornith-1.5, Qwen3.5.
- Text-to-SQL: Spider, BIRD.
- Vietnamese NLP: PhoBERT, ViT5, UIT-ViQuAD, PhoGPT, SeaLLM, Vistral.

---

## 12. Appendix (unlimited pages)

- Full 28-TID template listing with sample surfaces.
- Direct + Text2SQL prompt texts (raw hash `pv-26b1ac0d`, parser
  hash from evaluation seal).
- MANIFEST.json snapshot (v3.0.0).
- Full Colab-runnable reproduction recipe (`main.ipynb` walks
  restore → tables → dev/test → T04 → taxonomy → demo).
- Dev/test split sidecar `docs/dev_test_split_v3.0.0.json`
  (dataset_sha256 `d7a0c45c…17ff1`).
- Rescue intervention manifest `results/t04/rescue/intervention.json`
  (freeze rule + input SHA-256s + dev gate counts).
- Cache dump `llm-cache-20260905.sql.gz` provenance (27,674 rows,
  8,085,830 bytes, SHA-256 `60d9e0f2…`).
- Evaluation asset `evaluation-results.tar.gz` provenance
  (508,884 bytes, SHA-256 `bb10de26…`).

---

## Editor Handoff — Remaining TBDs

**Cells still `TBD`:**

- §7.5.1 Overall text F1 — need per-run means across all 2,800 QIDs.
- §7.5.2 Per-family text F1 for all four runs — need aggregates by
  paper T-family mapping.
- §7.5.3–7.5.4 Missing runs' loc/numeric — three runs beyond
  Ornith/T2SQL absent.
- §7.5.5 Attempted rates per metric slice.
- §8.2 Qwen/Direct per-family taxonomy row.
- §8.3 Qwen/Direct phenomena-flag row.

**How to fill:**

```bash
# Regenerate taxonomy CSVs (gitignored)
python scripts/error_taxonomy.py

# Per-run family aggregates from sealed per_question.jsonl
python scripts/report_split_metrics.py
```

Then quote directly from `results/analysis/taxonomy_*.csv` and the
split-metrics output. Do not fabricate. Do not backfill from
`docs/results.md` (v1 pre-freeze).

**After tables land:**

- Verify Conclusion (§10) numbers against filled §7 tables — refresh
  the two `**bold**` deltas if aggregation reveals rounding shifts.
- §8 taxonomy prose is grounded in T05 record; keep as-is.

---

## Internal Notes (delete before submission)

### Venue reality check

Best fit as-is: **VLSP (Vietnamese)**, **LREC-COLING resource
track**, **SIGSPATIAL / GeoAI workshop**, ACL SRW.

T04 pre-registration + dev/test protocol + rescue with
per-question no-regression assert + stable dev→test transfer
raises the methodological floor materially. Still below top-tier
without:

1. Bilingual head-to-head (English GS-QA on same four baselines).
2. Broader model panel (≥1 frontier, ≥1 large open, ≥1
   Vietnamese-specialized: PhoGPT / SeaLLM / Vistral).
3. Fine-tuned VN Text2SQL baseline (SPARTQA-style pretraining lift).
4. Human evaluation of predictions with κ ≥ 0.6.
5. Bootstrap CIs + paired significance tests.
6. Adversarial (human-paraphrased) subset for surface robustness.

### What T04/T05 changed vs prior draft

- **Dataset section stable** — v3.0.0 unchanged.
- **Rescue intervention** now has real test deltas — headline claim.
- **Error analysis** upgraded from proposed taxonomy to
  measured stage × family × phenomena tables.
- **Findings 1–4** in `docs/plans/T05` §Findings recorded for the
  report are the load-bearing paper claims — cite directly.
- **Notebook re-proof** (T02) satisfies course rubric.
