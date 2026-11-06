# Custom modifications in the current working tree

## Scope and comparison basis

This inventory records modifications visible in the working tree relative to the checked-out `HEAD` at documentation time. The tree was substantially dirty: many tracked files were modified, and new files were untracked. Git status cannot establish who authored a change, whether it is intended for release, or whether every modified file belongs to one feature. Do not reset, revert, or overwrite these files based only on this inventory.

This is a comparison to local `HEAD`, not a verified comparison to the latest upstream branch or to the live commercial deployment. The GitHub project README itself describes a periodic snapshot.

## Functional additions visible in the tree

### Dataset import, lookup, and administration

New/untracked files:

- `app/admin/dataset/DatasetAdmin.tsx`
- `app/admin/dataset/page.tsx`
- `app/api/admin/dataset/route.ts`
- `app/api/dataset/overview/route.ts`
- `app/api/dataset/topic/route.ts`
- `lib/dataset/import-year.ts`
- `lib/dataset/lookup.ts`
- `lib/dataset/service.ts`
- `scripts/import-year.ts`
- `scripts/regression/dataset-lookup.test.ts`

These add streaming year-file import, SQLite schema, key-based lookup for all 13 topics, dataset stats, an admin page, and raw dataset API routes. Current local data includes untracked `dataset/year-1991.jsonl`; it is a large local artifact, not proof that the released multi-year corpus or an imported DB is present.

### Explicit translation

New/untracked files:

- `app/api/dataset/translate/route.ts`
- `lib/translation/service.ts`
- `lib/translation/ziwei-dictionary.ts`
- `scripts/regression/translation-cache.test.ts`

These add a SQLite translation cache, source-hash versioning, dictionary normalization, explicit translation route, retryable model fallback, and original-source fallback behavior.

### Generative AI endpoint

`app/api/interpret/route.ts` is untracked in this tree. It implements a Gemini chat/interpretation endpoint with SSE output and model fallback. This conflicts with README statements that `/api/interpret` is absent in the open-source project, but is evidence of a route in this working tree.

### Display and name localization helpers

New/untracked files:

- `lib/ziwei/display-text.ts`
- `lib/ziwei/palace-name.ts`
- `lib/ziwei/star-name.ts`

They support the display-language changes also visible in modified UI and domain files. Inspect consumers and tests before altering source-level star names; chart logic, data matching, and display labels can use different representations.

## Tracked files changed relative to `HEAD`

The current status showed these tracked paths as modified:

- Root/runtime configuration: `.env.example`, `.gitignore`, `next-env.d.ts`, `package.json`, `package-lock.json`, `tsconfig.json`.
- App pages: `app/chart/page.tsx`, `app/globals.css`, `app/heming/page.tsx`, `app/knowledge/[star]/[topic]/page.tsx`, `app/knowledge/page.tsx`, `app/layout.tsx`, `app/page.tsx`, `app/preview/page.tsx`, `app/privacy/page.tsx`, `app/terms/page.tsx`.
- Library UI: `app/library/LibrarySearch.tsx`, `app/library/[book]/[chapter]/page.tsx`, `app/library/[book]/page.tsx`, `app/library/page.tsx`, `app/library/search/page.tsx`.
- Components: `components/AnnouncementModal.tsx`, `components/BirthForm.tsx`, `components/ChartBoard.tsx`, `components/ChartSummary.tsx`, `components/ChatPanel.tsx`, `components/FamousPersonCard.tsx`, `components/InsightPanel.tsx`, `components/PalaceCell.tsx`, `components/PatternsCard.tsx`, `components/ScrollIntro.tsx`, `components/ShareCardCanvas.tsx`, `components/ShareModal.tsx`, `components/StarDetailPanel.tsx`, `components/TimeNav.tsx`.
- Lecture documentation: `docs/tianji-ziwei/2上.md`, `3上.md`, `4上.md`, `5上.md`, `6上.md`, `7上.md`, `8上.md`, `9上.md`, `10上.md`, `11上.md`, `12上.md`, `13上.md`, `14上.md`, `15上.md`, `19上_补充1.md`, `24下_补充2.md`, and `docs/tianji-ziwei/README.md`.
- Classic/Ni Haixia/SEO material: `lib/classics/data/gusuifu.ts`, `lib/classics/data/quanji.ts`, `lib/classics/data/quanshu.ts`, `lib/nihai/index.ts`, `lib/nihai/renji.ts`, `lib/nihai/tianji.ts`, `lib/seo/knowledge.ts`.
- Chart and knowledge domain: `lib/ziwei/algorithm.ts`, `lib/ziwei/cities.ts`, `lib/ziwei/constants.ts`, `lib/ziwei/db-analysis.ts`, `lib/ziwei/famous.ts`, `lib/ziwei/heming-knowledge.ts`, `lib/ziwei/lunar-solar.ts`, `lib/ziwei/patterns.ts`, `lib/ziwei/school-config.ts`, `lib/ziwei/school-data.ts`, `lib/ziwei/share.ts`, `lib/ziwei/shier-shen.ts`, `lib/ziwei/sihua.ts`, `lib/ziwei/true-solar-time.ts`, `lib/ziwei/types.ts`.
- Regression tests: `scripts/regression/birthform-birthinfo-consistency.test.ts`, `scripts/regression/clock-adjust.test.ts`, `scripts/regression/form-true-solar-day.test.ts`, `scripts/regression/natal-dual-sihua-same-palace.test.ts`, `scripts/regression/neutral-naming.test.ts`, `scripts/regression/pan-integrity.test.ts`, `scripts/regression/patterns-sihua-bonus.test.ts`.

Observed themes include Vietnamese UI/name localization, chart-engine and school-rule changes, data/knowledge updates, documentation/transcript changes, and tests. The exact intent of each edit must be established from its diff and related code before making follow-up changes.

## Additional untracked worktree artifacts

Besides the functional source additions above, the following untracked files were present:

- `chinese.txt`, `remaining.txt`
- `common_replace.py`, `dump_remaining.py`, `star_global.py`
- `translate_announcement.py`, `translate_birthform.py`, `translate_insight.py`, `translate_stars.py`
- `dataset/year-1991.jsonl`

They are not declared as package scripts or runtime dependencies. Treat the Python/text utilities as workbench artifacts unless their owner confirms otherwise. Protect the large dataset file and avoid exposing its contents.

## Differences from the README's described open-source snapshot

The README says the open-source release excludes the interpretation database, AI prompts, and all `/api/*` backend routes. In this working tree:

- Dataset SQLite importer, lookup/service, admin API/UI, overview/topic/translate API routes exist as untracked files.
- `/api/interpret` exists as an untracked Gemini endpoint.
- `InsightPanel` is modified to retrieve dataset topics, while also retaining a separate AI chat path.
- Local environment examples point at machine-specific dataset paths.

These are local working-tree differences, not evidence that the upstream release or commercial deployment has the same implementation. The README's v4.0 scale and release metadata do not establish which year files are locally installed.

The README names `ca268829f7a9` and `ANALYSIS_CONTENT_VERSION`; source search did not find a runtime versioning implementation. The translation cache's source hash is the only explicit content-sensitive cache key observed; it does not version the overall chart engine or dataset release.
