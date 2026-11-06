# AI and interpretation pipeline

## Keep the three paths separate

There are three distinct sources of interpretation-like UI content:

1. **Dataset retrieval**: prewritten text indexed by chart key and topic. No model is involved.
2. **Dataset translation**: explicit translation of retrieved dataset text. Gemini is intended only as a translator; the original dataset remains unchanged.
3. **Generative AI chat**: freeform Gemini answers, including prompts automatically created when a palace or Four Transformation is selected. This path can produce new astrology interpretation.

The separate paths are visible in [`components/InsightPanel.tsx`](../../components/InsightPanel.tsx), [`app/api/dataset/overview/route.ts`](../../app/api/dataset/overview/route.ts), [`app/api/dataset/topic/route.ts`](../../app/api/dataset/topic/route.ts), [`app/api/dataset/translate/route.ts`](../../app/api/dataset/translate/route.ts), and [`app/api/interpret/route.ts`](../../app/api/interpret/route.ts).

## Dataset display path

On mount, `InsightPanel` requests `/api/dataset/overview`. Each topic button clears the prior dataset topic and fetches either overview or `/api/dataset/topic`; the selected text is rendered as the current dataset pane. Dataset contents are not appended to the AI message state. The original Chinese dataset text is authoritative and is displayed without automatic translation.

The topic buttons cover the same 13 dimensions as the database. Only overview currently displays a translation button. The topic API supports translation for any valid dimension if called directly, but the UI does not currently offer those other translation actions.

## Translation path

[`lib/translation/service.ts`](../../lib/translation/service.ts) implements:

1. SHA-256 hashing of the original Chinese source text.
2. A cache lookup keyed by `(dataset_id, section, language, source_hash)`.
3. Deterministic star/palace normalization through [`ziwei-dictionary.ts`](../../lib/translation/ziwei-dictionary.ts) if there is no cache hit.
4. Gemini translation with a strict Vietnamese system instruction that says to translate only, preserve Markdown/order/examples, and not interpret, summarize, add, remove, or rewrite content.
5. Save of the translated output in the same SQLite database's `dataset_translations` table.

The table stores `id`, `dataset_id`, `section`, `language`, `source_hash`, `translated_content`, `provider`, `model`, `created_at`, and `updated_at`, with a unique cache key on `(dataset_id, section, language, source_hash)`. `dataset_id` is the generated chart dataset key, `section` is the requested topic, and the current language is `vi`. Old rows in the prior `translations` table are copied once into the new table as legacy-provider entries; the old table is retained as `translations_legacy`. An in-process map deduplicates concurrent requests for the same key/section/language/source hash; it is not a distributed lock across server instances.

`GEMINI_TRANSLATION_MODEL`, when set, is tried first, then `models/gemini-3.5-flash-lite`, `models/gemini-3.5-flash`, and `models/gemini-2.5-flash`. Model fallback is attempted for 404, 429, 503, quota/resource exhaustion, rate-limit, and service-unavailable errors. If translation, cache lookup, or persistence fails, the service returns original Chinese content. The API returns that content with HTTP 200, `translated: false`, and `fallback: true`; dataset retrieval failures remain separate API errors.

The UI currently expects `translated: true` and treats the fallback object as invalid. The Chinese dataset text is not cleared while translation is pending, so a failed translation does not erase it, but the UI should be updated to render fallback content as a successful source-text response. Do not claim that a prompt can guarantee 100% semantic equivalence; the implementation's prompt is a constraint, not a proof of translation fidelity.

### Dictionary caveat

The fixed mapping includes the 14 major stars and 12 palace names (for example `紫微` to `Tử Vi`, `命宫` to `Mệnh cung`). The implementation should be reviewed against the requirement that replacements apply only to whole terms: Chinese text has no whitespace token boundaries, and a substring replacement can match a term embedded in a longer string. Preserve the source text and test examples carefully before changing this behavior.

## Generative chat path

The current `InsightPanel` uses `/api/interpret` for:

- user-entered follow-up questions;
- automatic prompts when a palace is selected;
- automatic prompts when a Four Transformation is selected.

It sends the chart JSON and conversation messages to the API. The route's system instruction requests Vietnamese Zi Wei interpretation and continued conversation. This is a true generation path, not a translation-only service, and it can generate new astrological content. It must remain distinct from the dataset translation cache and must not be used to populate or rewrite dataset rows.

Operational concerns in the route:

- It logs the complete chart JSON on model attempts.
- It lists available Gemini models at module startup if the key exists.
- It depends on API key/model availability and may call several models on retryable errors.
- The UI still has a missing-route message for `/api/interpret`, which is stale for this checkout.

## Source-of-truth invariants

- Never mutate the Chinese dataset row as part of translation.
- Never replace pure `/api/dataset/overview` or `/api/dataset/topic` retrieval with a model call.
- Only treat cached text as valid for the exact source hash.
- Keep translation and generative chat in different routes/services.
- Preserve full source text and return it on model-unavailability fallback.
- Do not silently present generated text as authoritative dataset content.
- Treat user chart data as potentially sensitive; avoid logging full chart payloads.

## Observability

The service logs `CACHE HIT`, `CACHE MISS`, `TRANSLATION SAVED`, `MODEL FALLBACK`, and `ALL MODELS FAILED`. The translation endpoint logs dataset key, topic, translation source, and Chinese/Vietnamese character counts. Review whether logs should include provider errors or model/usage metadata in production, and avoid logging content or API keys.
