# Open questions and maintenance priorities

This file separates observed implementation gaps from recommendations. Recommendations are not known commitments or an existing project roadmap.

## Confirmed implementation questions and risks

### 1. Translation fallback remains Chinese in the UI

`/api/dataset/translate` returns original Chinese text with `translated: false` and `fallback: true` when translation is unavailable. `InsightPanel` keeps that content visible and allows retry, but the user may need clearer UI feedback that the original source is being shown instead of Vietnamese.

### 2. Dataset language coverage is incomplete in the UI

The translation API accepts all supported topics, but the UI adds a translation button only to overview. Other 12 topic buttons display the dataset's original text. Decide whether this is intentional or whether translation controls should be exposed per topic.

### 3. Generative interpretation conflicts with translation-only policy

The explicit translation service uses a translation-only prompt, but `InsightPanel` still sends follow-up, selected-palace, and selected-Four-Transformation prompts to `/api/interpret`. That endpoint is explicitly an interpretation assistant. Clarify whether the no-new-astrology-content constraint applies only to dataset content or to all chart-analysis UI. Do not conflate this route with the translation service.

### 4. Sensitive chart data is logged

`/api/interpret` logs the full serialized chart before each model attempt. This can expose birth details and chart data in server logs. Decide the data-minimization/logging policy and remove or redact payload logging before production use. Also review logging of provider error objects in translation retry warnings.

### 5. Gemini model discovery runs at route module initialization

`/api/interpret` calls `client.models.list()` at module load when `GEMINI_API_KEY` exists. This creates an external request and logs the available models outside an explicit user request. Decide whether model discovery is needed; if retained, move it behind an explicit diagnostic/health mechanism and avoid running it during build/module collection.

### 6. API/UI route mismatches

- `app/heming/page.tsx` references `/api/heming`, but no such route exists.
- `InsightPanel`'s missing-route UI says `/api/interpret` is absent, although it exists in the current worktree.
- README says backend APIs are excluded, while current local untracked API routes exist.

Clarify release scope, then align copy, routes, and documentation.

### 7. Dataset environment is machine-specific

`ZIWEI_DATASET_SOURCE_DIR` and `ZIWEI_DATASET_DB_PATH` examples use Windows-specific paths, and the admin route has a hard-coded Windows fallback. A deployment needs a persistent writable database path and a mounted/source-file strategy. Define those explicitly for development, CI, and production.

### 8. Admin UI and development authentication differ

The API bypasses token authentication outside production, but the admin client appears to disable status loading until a token is provided. Confirm desired development UX. Production deployment must set `ZIWEI_DATASET_ADMIN_TOKEN`; missing configuration returns 503.

### 9. SQLite and compressed-import runtime compatibility

CI uses Node 22 but does not test SQLite lookup, zstd decompression of an actual compressed year, or import into the deployment database. Establish a minimum Node version and add an integration test on that version. The importer currently uses synchronous SQLite with a transaction across the full streamed import.

### 10. Dataset completeness and provenance

README release numbers describe 1950–2026 and 77 year files. Locally, an untracked 1991 JSONL file was observed, and no DB was listed in `dataset/`. There is no import manifest, dataset-version column, source-file checksum, or import timestamp. Determine which years are expected in each environment and how to identify/roll back a partially or incorrectly imported release.

### 11. Version identifiers are documentation-only in this source

`ca268829f7a9` and `ANALYSIS_CONTENT_VERSION` were found in README text, not in runtime source. There is no full dataset release registry. Translation cache identity is based on dataset key, topic, and content hash. Decide whether reproducible engine/data release versioning is needed.

### 12. Dictionary replacement semantics need validation

The translation dictionary is deterministic, but Chinese text does not provide whitespace word boundaries. Verify whether the implementation truly satisfies the documented whole-term requirement when a mapped term is embedded in a longer string. Add examples around overlapping terms and preserve source text exactly.

### 13. Existing scoring/rating behavior is not established

There are pattern levels/conditions and interpretive knowledge modules, but the inspected source did not establish a commercial-style normalized radar score formula for career, wealth, emotion, personality, and health. Do not infer that a chart rating exists just because pattern rules have qualitative levels. Locate an authoritative specification or implementation before reproducing a score.

### 14. Localized display names and engine names can diverge

The current diff changes chart/domain name data while new display conversion helpers are untracked. Determine whether Chinese source names or Vietnamese display names are canonical in each boundary. Dataset keys are unaffected by star names, but star matching, pattern detection, SEO/static data, and UI translation can be affected.

### 15. Worktree provenance is unclear

The worktree has extensive tracked modifications and untracked additions, including large local data and one-off translation scripts. Their authorship/release intent is unknown. Review diffs individually; do not clean up or overwrite them as part of unrelated maintenance.

## Suggested roadmap (recommendations only)

1. **Stabilize behavior contracts:** document and test dataset-only, translation, and generative-chat responsibilities; fix fallback handling and stale route copy.
2. **Reduce data exposure:** stop logging chart JSON and review all request/provider logs.
3. **Make deployment reproducible:** configure persistent DB and dataset source paths; test migration, import, backup, and restart on the actual Node runtime.
4. **Add API integration tests:** cover valid/malformed requests, missing keys, 13 topics, cache hit, changed source hash, model exhaustion fallback, and admin auth behavior.
5. **Resolve content versioning:** decide whether dataset release/provenance metadata is required and how translation cache invalidation should work.
6. **Reconcile docs/release scope:** distinguish upstream open-source behavior from this custom checkout and keep README, UI error messages, and route inventory consistent.

No existing issue tracker or committed roadmap was consulted; these priorities follow from observed code paths and gaps only.
