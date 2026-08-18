# 07 — Do NOT break

Small, high-signal list. Read this file *every* task. If a proposed change touches any item below, stop and re-plan.

## Schema / DB

- Do NOT drop or rename any column on `tmgmt_capi_request_processor`, especially: `file_upload_status`, `file_upload_attempts`, `capi_response`, `file_data`, `source_site`, `updateid`, `statuscode`, `status`.
- Do NOT drop or rename any index — several queries are named after their backing index.
- Do NOT edit historical `tmgmt_contentapi_update_9XXX()` hooks.
- Do NOT backfill `source_site` on existing NULL rows.
- Do NOT re-use hook number `9104` (skipped gap).
- Do NOT remove `uq_dedup` on `tmgmt_contentapi_bundle_queue`.

## Concurrency / multi-container

- Do NOT store per-request/per-job memory (files, tokens, manifests) in `\Drupal::state()` for cross-container consumption. Use the DB (`file_data` pattern).
- Do NOT remove the atomic `UPDATE … WHERE file_upload_status = 'PENDING'` guard.
- Do NOT bypass `AutoBundleFlusher`'s `@lock` acquisition.

## Cross-environment

- Do NOT run reconciliation, upload, import, or status-update queries without the `source_site = current OR source_site IS NULL` predicate on `tmgmt_capi_request_processor`.

## Redelivery

- Do NOT remove or gate the FINISHED-job redelivery check in `CapiDataProcessor` (~L472–L595). Regression symptom: late translations vanish.

## Orphan handling

- Do NOT remove the two orphan-marking paths (CapiDataProcessor ~L377; QueueOperations ~L1231). Regression symptom: zombie rows churn queues.

## Status columns

- Do NOT confuse `status` (row workflow) with `statuscode` (CAPI-side state). They are separate columns with overlapping value names.
- Do NOT introduce a new status literal without adding a constant on `CapiDataProcessor` and auditing every filter that could reject it.
- Do NOT branch on `statuscode = 'CREATED'` without also considering `file_upload_status`.

## API client

- Do NOT hand-edit `src/Swagger/Client/**`. Regenerate.

## TMGMT / upstream

- Do NOT edit `web/modules/contrib/tmgmt/**` (TMGMT core).
- Do NOT edit `web/modules/contrib/lionbridge_translation_provider/**` (upstream mirror). The live path is `web/sites/default/modules/contrib/lionbridge_translation_provider/**`.

## PHP / Drupal support

- Do NOT add PHP 8.2+ syntax if a PHP 8.1-only user could hit it (readonly-class, `never` in interface positions used by decorators, etc.). Confirm before using.
- Do NOT reintroduce `#[\ReturnTypeWillChange]`.
- Do NOT use `array $x = NULL` (bare); use `?array $x = NULL`.
- Do NOT pass `NULL` to `explode`, `substr`, `implode`, `strlen`, etc. — use `?? ''` at the boundary.

## Continuous jobs

- Do NOT mutate continuous-job items inside a code path that itself listens on the same events without entering `ContinuousReQueueSuppressor` — infinite re-queue risk.

## Auto-Bundle

- Do NOT introduce non-determinism (unsorted arrays, timestamps, random) into `AutoBundleGroupKeyBuilder`. Group keys must be reproducible.
- Do NOT let `AutoBundleTriggerEvaluator` fire past `cap_max_items` / `cap_max_words`.
- Do NOT flush a bundle without holding `@lock` scoped to that group_key.

## Logging / UX

- Do NOT log the CAPI token, provider credentials, or JWT.
- Do NOT surface raw CAPI response bodies to end users; strip / summarize.
- Do NOT delete the `Reports → TMGMT_CONTENTAPI` operator surface — it's the recovery UI.
