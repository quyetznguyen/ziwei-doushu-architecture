# Project overview

## What this repository contains

This is a Next.js application and Zi Wei Dou Shu charting engine. The checked-in project describes itself as an open-source engine/demo, while the current working tree also contains local additions for a SQLite interpretation dataset, its administration UI, and Gemini-backed translation/chat endpoints. The checked-out source and the README therefore do not describe the same runtime surface in every respect; see [06_CUSTOM_MODIFICATIONS.md](./06_CUSTOM_MODIFICATIONS.md).

The application has three distinct kinds of content:

1. **Calculated chart data** produced by `generateChart()` from Gregorian birth data. The engine is in [`lib/ziwei/algorithm.ts`](../../lib/ziwei/algorithm.ts); public chart types are in [`lib/ziwei/types.ts`](../../lib/ziwei/types.ts).
2. **Deterministic local knowledge** such as star profiles, pattern rules, compatibility knowledge, and classic texts. These are source-controlled modules and static data, not Gemini output.
3. **Per-chart interpretation text** retrieved from SQLite in the current worktree. Dataset retrieval is separate from the chart calculation. Gemini is used by the translation path only when explicitly requested and uncached; the freeform chat route is a separate generative path.

## Runtime shape

```text
BirthForm
  -> formToBirthInfo()
  -> generateChart()
     -> ZiweiChart { birthInfo, lunarInfo, palaces, daXians, ... }
  -> ChartBoard + TimeNav + InsightPanel
     -> POST /api/dataset/overview or /api/dataset/topic
        -> buildDatasetKey(chart.birthInfo)
        -> SQLite chart_interpretations lookup
     -> explicit overview translation button
        -> POST /api/dataset/translate
        -> translations cache -> Gemini fallback models if uncached
     -> separate chat / palace / Four-Transformation prompts
        -> POST /api/interpret -> Gemini response stream
```

The charting route is client-side: [`app/chart/page.tsx`](../../app/chart/page.tsx) calls `generateChart()` directly after the form submits. Dataset and AI APIs execute server-side and declare the Node.js runtime because they use SQLite and/or Node APIs.

## Birth data to chart

- [`components/BirthForm.tsx`](../../components/BirthForm.tsx) collects Gregorian year/month/day, gender, clock time, an optional place/longitude, and an unknown-time option.
- [`lib/ziwei/share.ts`](../../lib/ziwei/share.ts) converts the form into `BirthInfo`. With a known time it calculates the true-solar hour branch from clock time and longitude, and resolves late-Zi day rollover. Unknown time is represented as branch `0`.
- [`lib/ziwei/algorithm.ts`](../../lib/ziwei/algorithm.ts) uses `iztro` and `lunar-javascript` and applies repository-specific school settings and patches. It returns a `ZiweiChart`; it does not call the dataset or Gemini.
- The dataset key uses only `chart.birthInfo.year`, `.month`, `.day`, `.hour`, and `.gender`. `hour` is the effective branch index `0..11`, not a wall-clock hour. Key generation is in [`lib/dataset/lookup.ts`](../../lib/dataset/lookup.ts).

The chart engine has important conventions for true solar time, late Zi, leap months, school-specific star placement, transformations, and time-flow calculations. Read [03_DATASET_ARCHITECTURE.md](./03_DATASET_ARCHITECTURE.md) before changing the key or data lookup, and consult the specific engine modules before changing chart inputs or outputs.

## Interpretation behavior

`InsightPanel` initially loads the original dataset overview from `/api/dataset/overview`. Its topic controls replace one dataset topic with another; they do not append dataset topics to the chat thread. The dataset contains 13 dimensions: overview, personality, love, career, wealth, health, family, children, move, friends, home, spirit, and parents.

The overview's **Dịch sang tiếng Việt** action calls `/api/dataset/translate`. Other topic buttons currently show raw dataset text and do not have a translation action. The AI chat and automatic prompts for selected palaces/Four Transformations are separate from dataset retrieval and call `/api/interpret`; this route can generate interpretation rather than merely translate. Do not describe the whole application as AI-free or as translation-only.

The source also contains deterministic interpretive knowledge, including [`lib/ziwei/db-analysis.ts`](../../lib/ziwei/db-analysis.ts), [`lib/ziwei/patterns.ts`](../../lib/ziwei/patterns.ts), [`lib/ziwei/heming-knowledge.ts`](../../lib/ziwei/heming-knowledge.ts), [`lib/ziwei/famous.ts`](../../lib/ziwei/famous.ts), and [`components/StarDetailPanel.tsx`](../../components/StarDetailPanel.tsx). Classic text data is under [`lib/classics/data/`](../../lib/classics/data/). These are not the SQLite per-chart interpretation dataset.

## Technology and local commands

- Next.js App Router, React, TypeScript, Tailwind CSS, and Framer Motion.
- Chart calculation uses `iztro` and `lunar-javascript`.
- The current dataset/translation implementation uses Node's built-in `node:sqlite` synchronous API; no SQLite npm package is declared.
- Gemini client dependency: `@google/genai`.
- CI is configured for Node 22 and runs `npm ci`, `npm run typecheck`, and `npm test` in [`.github/workflows/ci.yml`](../../.github/workflows/ci.yml).
- Local commands are declared in [`package.json`](../../package.json): `npm run dev`, `npm run build`, `npm run typecheck`, and `npm test`.

The CI workflow does not import the compressed year dataset. Runtime support for Node's Zstandard decompressor therefore needs separate verification before running the admin import flow on a deployment target.

## Project boundaries and evidence

The README documents a commercial service and a periodic open-source snapshot. It also says the commercial UI, AI interpretation, interpretation database, and backend routes are not part of the open-source release. In this working tree, new untracked source files implement several of those surfaces. Treat repository files as evidence of this checkout; treat README and release claims as statements about their stated release/service, not proof that the corresponding data or route is present locally.

The README mentions website content identifier `ca268829f7a9` and `ANALYSIS_CONTENT_VERSION`; a source search found those identifiers in README text, not an implemented runtime version registry. Translation caching has its own source hash, but that is not a full dataset/content release versioning system.

For current gaps, risks, and recommendations, see [07_OPEN_QUESTIONS.md](./07_OPEN_QUESTIONS.md). These documents are a source-grounded handoff, not a claim that every modified file has been reviewed or that the current dirty worktree is a production deployment.
