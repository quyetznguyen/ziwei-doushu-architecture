# API architecture

## Route inventory

All routes below were present in the current working tree when documented. Several are untracked relative to `HEAD`.

| Method and path | Request | Behavior and response |
|---|---|---|
| `POST /api/dataset/overview` | `{ "chart": ZiweiChart-like object }` | Validates `chart.birthInfo`, computes the key, retrieves overview. Returns `{ datasetKey, overview, translated: false }`; no Gemini call. |
| `POST /api/dataset/topic` | `{ "chart": ..., "topic": DatasetDimension }` | Retrieves any of the 13 dimensions. Returns `{ topic, content }`. |
| `POST /api/dataset/translate` | `{ "chart": ..., "topic": DatasetDimension }` | Retrieves source text, returns cached translation or calls Gemini and caches it. Success returns `{ datasetKey, topic, content, translated: true, source }`. Translation failure returns HTTP 200 with `{ datasetKey, topic, translated: false, fallback: true, content: originalChinese, source: "fallback" }`. |
| `GET /api/admin/dataset` | Optional bearer token in production | Scans source directory and returns available years, imported years, database path, and record count. |
| `POST /api/admin/dataset` | `{ "year": integer }`; bearer token in production | Imports matching `year-XXXX.jsonl.zst` and returns updated state plus a message. |
| `POST /api/interpret` | `{ "chart": object, "messages": [{ "role": "user" \| "assistant", "content": string }] }` | Sends chart JSON and chat history to Gemini; success is an SSE response with text deltas and `[DONE]`. |

Each route uses the Node.js runtime where configured because the server code uses Node APIs/SQLite. Dataset endpoints use `NextResponse` JSON errors; `/api/interpret` has a custom SSE response for successful generation.

## Dataset route details

The overview route is intentionally a retrieval-only boundary. It uses `buildDatasetKey()` and `getOverview()` and does not import the translation service. The topic route restricts the topic to `DATASET_DIMENSIONS` and shares the same key construction/service behavior.

The translation route validates the same chart/topic shape, then:

1. Loads the original topic text from SQLite.
2. Calls `translateDatasetContent(datasetKey, topic, chineseContent)`.
3. Logs the key, topic, source (`cache` or `gemini`), and source/translated character counts.
4. Returns translated text, or original source text with a non-error fallback response when the translation service cannot translate or persist the result.

The UI accepts translated responses and the explicit original-text fallback. On fallback, it keeps the original dataset content visible and allows another translation attempt.

## Admin security and filesystem behavior

[`app/api/admin/dataset/route.ts`](../../app/api/admin/dataset/route.ts) skips authentication when `NODE_ENV !== 'production'`. In production, the route requires `ZIWEI_DATASET_ADMIN_TOKEN`, compares bearer token bytes with `timingSafeEqual`, returns 503 when token configuration is missing, and 401 for a mismatch.

The route reads the configured source directory, defaulting to `D:\Projects\data-ziwei-doushu\ziwei-interp-v4-all`, and imports files from that directory. The default is not portable. The client component currently appears to require a token before enabling status loading even in development, which is inconsistent with the API's development bypass.

## Gemini interpretation endpoint

[`app/api/interpret/route.ts`](../../app/api/interpret/route.ts) is separate from the dataset API:

- It accepts the complete chart object and message history.
- It builds a text prompt containing `JSON.stringify(chart)` and the conversation.
- It calls a preferred `GEMINI_MODEL` first, then a fallback model list.
- Retryable rate-limit/service-unavailable failures advance to the next model; other errors also proceed through the loop but do not use a distinct retry policy branch to stop.
- A successful text response is wrapped as SSE.
- The route logs the full chart JSON before each model request.
- It also calls `client.models.list()` at module initialization when an API key is configured; this can cause an external request during route module loading/build collection.

The system instruction for this route explicitly asks for Vietnamese astrology interpretation. It is not restricted to translation. For the distinct translation-only rules, see [05_AI_PIPELINE.md](./05_AI_PIPELINE.md).

## Missing route references and stale release documentation

The current source has no `/api/heming` or `/api/generate` route. `app/heming/page.tsx` references `/api/heming`; the chart UI has error copy that still says `/api/interpret` is absent, although that route exists in this working tree. README text states that backend routes are not included, which may describe the intended upstream release but does not match the current local working tree.

## Error behavior and validation notes

- Malformed JSON and invalid request shapes return 400.
- Valid keys with no dataset row/topic return 404.
- Database/path/read failures return 500 with a JSON error.
- Gemini model exhaustion in translation is intentionally not an HTTP 500; other translation errors can still return 500.
- No auth is applied to regular dataset read/translation routes in the source shown.
- Rate limiting, quotas, request-size limits, user identity, and privacy retention policies are not implemented in these route files.

Do not expose chart payloads or credentials in logs. The `/api/interpret` chart logging is a concrete privacy/operations concern to resolve before public deployment.
