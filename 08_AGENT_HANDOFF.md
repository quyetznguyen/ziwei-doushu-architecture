# Agent handoff

Read this before modifying the project. The most important rule is to distinguish chart calculation, deterministic/local knowledge, raw dataset retrieval, Gemini translation, and freeform Gemini interpretation. They are separate systems.

## Fast map

- Birth input and validation: [`components/BirthForm.tsx`](../../components/BirthForm.tsx)
- Form normalization, true-solar hour branch, and late-Zi day rollover: [`lib/ziwei/share.ts`](../../lib/ziwei/share.ts)
- Chart calculation and patches: [`lib/ziwei/algorithm.ts`](../../lib/ziwei/algorithm.ts)
- Chart contract: [`lib/ziwei/types.ts`](../../lib/ziwei/types.ts)
- Chart and interpretation UI: [`app/chart/page.tsx`](../../app/chart/page.tsx), [`components/InsightPanel.tsx`](../../components/InsightPanel.tsx)
- Dataset key, all 13 dimensions, and SQLite reads: [`lib/dataset/lookup.ts`](../../lib/dataset/lookup.ts)
- Dataset import: [`lib/dataset/import-year.ts`](../../lib/dataset/import-year.ts), [`scripts/import-year.ts`](../../scripts/import-year.ts)
- Pure read APIs: [`app/api/dataset/overview/route.ts`](../../app/api/dataset/overview/route.ts), [`app/api/dataset/topic/route.ts`](../../app/api/dataset/topic/route.ts)
- Translation only: [`lib/translation/service.ts`](../../lib/translation/service.ts), [`app/api/dataset/translate/route.ts`](../../app/api/dataset/translate/route.ts)
- Generative chat/interpretation: [`app/api/interpret/route.ts`](../../app/api/interpret/route.ts)
- Deterministic knowledge: [`lib/ziwei/db-analysis.ts`](../../lib/ziwei/db-analysis.ts), [`lib/ziwei/patterns.ts`](../../lib/ziwei/patterns.ts), [`lib/ziwei/famous.ts`](../../lib/ziwei/famous.ts), [`lib/ziwei/heming-knowledge.ts`](../../lib/ziwei/heming-knowledge.ts), [`components/StarDetailPanel.tsx`](../../components/StarDetailPanel.tsx)

## Non-negotiable invariants

1. **Do not change dataset-key semantics casually.** It is `YYYY-MM-DD-hourIndex-gender` from effective `chart.birthInfo.year/month/day/hour/gender`; hour is a branch index `0..11`.
2. **Keep raw retrieval model-free.** `/api/dataset/overview` and `/api/dataset/topic` must return dataset text without calling Gemini.
3. **Do not mutate source content during translation.** Translation is explicit, cache identity includes SHA-256 of source, and fallback should preserve original content.
4. **Do not mix the translation cache with AI chat.** `/api/interpret` is generative; it can create new astrology language. The translation endpoint is constrained to translation.
5. **Keep data shapes aligned.** There are 13 topic fields in the SQLite table, topic routes, UI, and tests. Update them consistently if a dimension is added/renamed.
6. **Treat birth/chart values and model logs as sensitive.** The current interpretation route logs full chart JSON; avoid expanding that exposure.
7. **Protect the existing dirty worktree.** Many files are already modified and many new files are untracked. Inspect status/diffs before editing and never discard unrelated work.

## Safest change workflow

1. Check `git status --short` and inspect the target file/diff.
2. Read the relevant architecture note (`03` dataset, `04` APIs, `05` AI, `07` gaps).
3. Trace the full boundary: form -> `BirthInfo` -> chart/key -> route/service -> component. Avoid fixing only one side of a contract.
4. Make a narrow change using existing patterns and update directly related docs/tests.
5. Run the smallest relevant test, then typecheck/build as appropriate. Do not treat an aggregate test pass as covering compressed imports unless a compressed file was actually imported.
6. Report anything not validated, especially external Gemini behavior and deployment filesystem setup.

## Validation and current baseline

Declared commands:

```text
npm run typecheck
npm test
npm run build
```

CI uses Node 22 and runs typecheck plus the full test script. In the local verification performed while preparing this documentation, `npm test` **failed** at `scripts/regression/natal-dual-sihua-same-palace.test.ts`: actual star names were Chinese (`天机`, `太阴`) while the assertion expected Vietnamese (`Thiên Cơ`, `Thái Âm`). Tests earlier in the chained command passed; later tests were not run by that aggregate invocation because the command stopped at the failure. Treat this as a known baseline failure, not a failure caused by the documentation.

## Known blockers to resolve before production claims

- Translation fallback renders original Chinese; user-facing fallback notice could be clearer.
- `/api/heming` is referenced but absent.
- `/api/interpret` still has stale missing-route copy and logs chart JSON.
- Gemini model discovery is invoked at API module initialization.
- Admin's development token UX differs from the API's token bypass.
- Dataset paths are Windows-specific; local full dataset/database availability is not established.
- Compressed import and deployment runtime compatibility are not covered by CI.
- The local worktree's extensive changes have unclear authorship/release intent.

See [07_OPEN_QUESTIONS.md](./07_OPEN_QUESTIONS.md) before taking these on; several require a product/security decision rather than an assumed implementation.

## Documentation mirror automation

On this Windows workstation, a scheduled task named `ZiweiDocsAutoSync` watches `docs/agent-knowledge/**/*.md`, checking file hashes every 2 seconds. After a 10-second quiet period it mirrors changed files into the root of `D:\Projects\ziwei-doushu-architecture`, commits only managed Markdown paths, and pushes `main` without force. Failed syncs are retried after 30 seconds. Deletions are limited to paths recorded in `.agent-knowledge-sync.json`; unrelated files in the architecture repository are not staged. The source Git `post-commit` hook delegates to the same one-shot sync script.

- Install/start or repair: run `powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\manage-agent-knowledge-watcher.ps1 -Install` from the source repository.
- Stop/remove: use the same command with `-Uninstall`.
- One-shot manual sync: run `.\sync-to-agent.ps1`.
- Logs: `%LOCALAPPDATA%\ZiweiDocsAutoSync\sync.log`.

The watcher uses the destination repository's configured Git credential helper. It refuses to push to an unexpected remote, another branch, or a destination branch that has diverged from GitHub.
