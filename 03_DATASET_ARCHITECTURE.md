# Dataset architecture

## Dataset contract

The current importer and lookup code use one row per chart key in SQLite. The expected JSONL object has these metadata fields:

| Field | Meaning in the importer |
|---|---|
| `k` | Dataset key; primary key in SQLite. |
| `y`, `m`, `d` | Gregorian date; `y` must match the source filename's year. |
| `h` | Zi Wei hour-branch index, integer `0..11`. |
| `g` | `male` or `female`. |
| `ms` | Nullable string metadata. |
| `t` | Map of the 13 dimension character counts; stored as JSON text in SQLite. |
| `c` | Map of the 13 dimension contents. |

The 13 required `c`/`t` keys are `overview`, `personality`, `love`, `career`, `wealth`, `health`, `family`, `children`, `move`, `friends`, `home`, `spirit`, and `parents`. The implementation defines them in [`lib/dataset/lookup.ts`](../../lib/dataset/lookup.ts).

The `chart_interpretations` table stores `k` as `TEXT PRIMARY KEY NOT NULL`, metadata columns, `t TEXT NOT NULL`, and one `TEXT NOT NULL` column for every dimension. All dimension content is returned by `lookupDataset()`. The translation service creates `dataset_translations` on demand, with a unique key over dataset ID, section, language, and source hash.

## Key generation and lookup

`buildDatasetKey(chart)` reads exactly:

```text
chart.birthInfo.year
chart.birthInfo.month
chart.birthInfo.day
chart.birthInfo.hour
chart.birthInfo.gender
```

It formats the date as zero-padded `YYYY-MM-DD` and appends the numeric branch index and gender, for example `1991-04-03-5-female`. It does not use name, location, raw clock hour/minute, timezone, true-solar offset, lunar date, or any chart palace/star field. The `birthInfo` values are the effective values the chart engine received/returned; the caller must therefore supply the correctly adjusted chart. Do not silently substitute wall-clock hour for `hour`.

[`lookupDataset()`](../../lib/dataset/lookup.ts) opens the configured SQLite file, performs a parameterized lookup by `k`, reads all 13 dimensions, validates each returned dimension is text, and closes the database in `finally`. Missing key returns `null`; missing configured database throws. The service layer in [`lib/dataset/service.ts`](../../lib/dataset/service.ts) gets `ZIWEI_DATASET_DB_PATH`, builds the key, then returns one topic or `null`.

## Import pipeline

[`scripts/import-year.ts`](../../scripts/import-year.ts) is the CLI. Its reusable importer is [`lib/dataset/import-year.ts`](../../lib/dataset/import-year.ts):

1. Accepts only filenames matching `year-XXXX.jsonl` or `year-XXXX.jsonl.zst`.
2. Streams the input line-by-line; `.zst` inputs are piped through Node's `node:zlib` Zstandard decompressor.
3. Parses and validates each JSON object, metadata ranges, year/key agreement, and presence/types for each of the 13 dimensions.
4. Creates the schema if needed, starts `BEGIN IMMEDIATE`, and upserts records using `ON CONFLICT(k) DO UPDATE`.
5. Commits after all lines succeed; rolls back and rethrows on failure. Blank lines and empty files are errors.

The importer does not fetch files from the internet. It expects source files to exist locally. The API importer creates the database parent directory before calling the importer; the CLI expects the database parent path to be usable.

## Administration surface

The admin page at [`app/admin/dataset/page.tsx`](../../app/admin/dataset/page.tsx) uses [`DatasetAdmin.tsx`](../../app/admin/dataset/DatasetAdmin.tsx) and `/api/admin/dataset` to display available years, imported years, database path, and record count. The route scans the configured source directory for `year-XXXX.jsonl.zst`; POST imports one requested year.

Configuration:

- `ZIWEI_DATASET_DB_PATH`: required database file path for dataset lookup, translation, and admin state.
- `ZIWEI_DATASET_SOURCE_DIR`: optional source directory for compressed year files. The route otherwise defaults to a hard-coded Windows path in `app/api/admin/dataset/route.ts`.
- `ZIWEI_DATASET_ADMIN_TOKEN`: required in production. In non-production the current route bypasses token authentication.

The admin UI and API are custom code in the current worktree; confirm filesystem permissions, deployment storage persistence, and production token provisioning before exposing the route.

## API read behavior

- `POST /api/dataset/overview` takes `{ chart }` and returns `{ datasetKey, overview, translated: false }`. It performs pure dataset lookup and never calls Gemini.
- `POST /api/dataset/topic` takes `{ chart, topic }` and returns `{ topic, content }` for a supported dimension.
- `POST /api/dataset/translate` takes `{ chart, topic }`, loads that topic's source text, then uses the translation cache or Gemini. See [05_AI_PIPELINE.md](./05_AI_PIPELINE.md).

Routes validate `birthInfo` field types/ranges and supported topic names, but lookup errors and missing records remain observable as HTTP errors (normally 404 for absent chart/topic and 500 for operational errors).

## Dataset scale versus local availability

The README describes the v4.0 release as 674,976 charts, 13 topics per chart, years 1950–2026, 77 year files, and a 56.6 GB uncompressed corpus. Those are release claims, not evidence that the full dataset is present in this checkout. At inspection time, only an untracked `dataset/year-1991.jsonl` (about 716 MB) was seen locally; no imported database was listed there. The admin page's external default source directory is also machine-specific. A chart whose year/key is absent from the database cannot get dataset text, even if the chart engine can calculate the chart.

## Versioning and mutation

The README names `ca268829f7a9` as the website content version and mentions `ANALYSIS_CONTENT_VERSION`, but neither identifier is implemented in the source files searched. There is no dataset-release/version metadata table in the current SQLite schema. Translation rows in `dataset_translations` are keyed by `(dataset_id, section, language, source_hash)`, where `source_hash` is SHA-256 of the original Chinese source text. This lets changed source text receive a distinct cached translation; it is not a release manifest for imported data.

Importer upserts replace all stored dimension values for a matching key. Import provenance, source-file checksum, version, and import timestamp are not stored. Before re-importing altered data, decide whether preserving old dataset snapshots or auditable provenance is required.

## Operational cautions

- `node:sqlite` is synchronous; each lookup opens and closes its database connection. Load/performance behavior should be measured before high-concurrency deployment.
- The translation service and dataset store share the same SQLite file. Backups and write permissions must cover both tables.
- Imports hold an immediate write transaction for the full input stream; large files can block other SQLite writers for a long time.
- Validate the deployed Node version supports both built-in SQLite and the Zstandard APIs used by the importer. CI runs Node 22 but does not exercise compressed import.
- Avoid printing dataset rows or the full 716 MB local JSONL to logs or chat.
