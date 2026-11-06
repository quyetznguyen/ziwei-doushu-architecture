# Directory structure

This guide describes the checked-out tree, including custom files in the current working tree. Some dataset/API files are untracked relative to `HEAD`; their presence here does not mean they are part of the upstream commit.

## Application routes

| Path | Responsibility |
|---|---|
| [`app/page.tsx`](../../app/page.tsx) | Main landing page. |
| [`app/chart/page.tsx`](../../app/chart/page.tsx) | Client-side birth form, chart generation, chart board, time navigation, and insight panel. |
| [`app/heming/page.tsx`](../../app/heming/page.tsx) | Compatibility page. It references `/api/heming`; no matching route exists in the current route tree. |
| [`app/knowledge/`](../../app/knowledge/) | Star/topic knowledge pages and listing pages. |
| [`app/library/`](../../app/library/) | Browse/search UI for bundled classic texts. |
| [`app/preview/`](../../app/preview/) | Preview route. |
| [`app/privacy/`](../../app/privacy/) and [`app/terms/`](../../app/terms/) | Policy pages. |
| [`app/admin/dataset/`](../../app/admin/dataset/) | Dataset administration page and client component in the current worktree. |
| [`app/api/interpret/route.ts`](../../app/api/interpret/route.ts) | Gemini interpretation/chat endpoint in the current worktree. |
| [`app/api/dataset/`](../../app/api/dataset/) | Raw overview/topic reads and explicit translation endpoint. |
| [`app/api/admin/dataset/route.ts`](../../app/api/admin/dataset/route.ts) | Dataset status and per-year import endpoint. |

The current app API routes do not include `/api/heming` or `/api/generate`, despite references to those paths in pages/comments or older integration context.

## Components

[`components/`](../../components/) contains the React UI. The chart flow is primarily:

- [`BirthForm.tsx`](../../components/BirthForm.tsx): form state and validation.
- [`ChartBoard.tsx`](../../components/ChartBoard.tsx), [`PalaceCell.tsx`](../../components/PalaceCell.tsx), and [`ChartSummary.tsx`](../../components/ChartSummary.tsx): chart rendering.
- [`TimeNav.tsx`](../../components/TimeNav.tsx): natal and time-flow view controls.
- [`InsightPanel.tsx`](../../components/InsightPanel.tsx): dataset topics, explicit overview translation, and separate AI chat.
- [`StarDetailPanel.tsx`](../../components/StarDetailPanel.tsx) and [`PatternsCard.tsx`](../../components/PatternsCard.tsx): star and pattern knowledge presentation.
- Theme, history, sharing, and announcement components support surrounding UX.

## Libraries

### `lib/ziwei/`

The chart domain: types, constants, algorithm, school settings/data/patches, lunar/solar-time handling, sharing/form conversion, time-flow calculations, transformation logic, patterns, star data, display-name conversion, and interpretation knowledge. Start with [`algorithm.ts`](../../lib/ziwei/algorithm.ts), [`types.ts`](../../lib/ziwei/types.ts), [`school-config.ts`](../../lib/ziwei/school-config.ts), and [`share.ts`](../../lib/ziwei/share.ts). This area is behavior-sensitive; see [08_AGENT_HANDOFF.md](./08_AGENT_HANDOFF.md).

### `lib/dataset/`

- [`lookup.ts`](../../lib/dataset/lookup.ts): 13 dimensions, key generation, SQLite lookup, and stats.
- [`service.ts`](../../lib/dataset/service.ts): database-path configuration and dimension getters.
- [`import-year.ts`](../../lib/dataset/import-year.ts): streaming JSONL/Zstandard import and SQLite schema/upsert logic.

### `lib/translation/`

- [`service.ts`](../../lib/translation/service.ts): translation cache, source hashing, in-flight deduplication, Gemini fallback, and original-source fallback.
- [`ziwei-dictionary.ts`](../../lib/translation/ziwei-dictionary.ts): deterministic star/palace term normalization before translation.

### Other domain libraries

- [`lib/classics/`](../../lib/classics/): classic book types, search helpers, and static text in `data/`.
- [`lib/nihai/`](../../lib/nihai/): Ni Haixia-related knowledge modules.
- [`lib/seo/`](../../lib/seo/): helpers for generated star/topic knowledge routes.
- [`lib/ziwei/db-analysis.ts`](../../lib/ziwei/db-analysis.ts): static star/topic interpretation data and topic mappings; distinct from the SQLite chart dataset.
- [`lib/ziwei/heming-knowledge.ts`](../../lib/ziwei/heming-knowledge.ts): compatibility-analysis knowledge; it does not imply a corresponding API route exists.

## Data, documentation, and tests

- [`docs/tianji-ziwei/`](../tianji-ziwei/): lecture/transcript-derived documentation and materials.
- [`docs/agent-knowledge/`](./): these maintenance and handoff documents.
- [`dataset/`](../../dataset/): the current worktree contains an untracked `year-1991.jsonl` file (about 716 MB). Do not commit or dump it as part of routine documentation work. No SQLite database was listed in that directory at the time these documents were written.
- External compressed year files are expected at `ZIWEI_DATASET_SOURCE_DIR`; the sample default path is machine-specific and is not portable.
- [`scripts/import-year.ts`](../../scripts/import-year.ts): CLI for import and lookup.
- [`scripts/regression/`](../../scripts/regression/): chart, boundary, dataset, and translation regression scripts. The aggregate command is `npm test`.
- Root configuration includes [`package.json`](../../package.json), `tsconfig.json`, Next/Tailwind/PostCSS config, [`.env.example`](../../.env.example), and [`.github/workflows/ci.yml`](../../.github/workflows/ci.yml).

The working tree also contains untracked one-off localization utilities and text files (`common_replace.py`, `dump_remaining.py`, `star_global.py`, `translate_*.py`, `chinese.txt`, and `remaining.txt`). They are not listed as package scripts or runtime dependencies; verify their intent before using or removing them.
