# 02 — Workflow & state machine

The `tmgmt_capi_request_processor` table drives the module's request lifecycle. It has **two distinct workflow columns** that are easy to confuse:

| Column | Semantics | Values (observed in code) |
|---|---|---|
| `status` | Row-level workflow inside Drupal (what the module is doing with this row) | `NEW`, `SCANNED`, `IGNORED`, `TO_PROCESS`, `IN_QUEUE`, `CANCELLED`, `IMPORTED`, `COMPLETED` |
| `statuscode` | CAPI-side request status (what Lionbridge says) | `CREATED`, `SENDING`, `SENT_TO_PROVIDER`, `IN_TRANSLATION`, `REVIEW_TRANSLATION`, `TRANSLATION_APPROVED`, `CANCELLED` |
| `file_upload_status` | Upload sub-state for `CREATED`/`SENDING` rows | `PENDING`, `UPLOADING`, `UPLOADED`, `FAILED` (plus `FILE_GENERATING`, `READY_FOR_UPLOAD` in some queries) |

Values live in constants on `Services/CapiDataProcessor.php` — search there before hardcoding a new one.

> Any transition changes below marked `INFERRED FROM CODE` should be re-verified with `grep_search` for the literal status string before you rely on them.

## Row lifecycle (`status` column) — INFERRED

```
        (row inserted)
              |
              v
           NEW / TO_PROCESS
              |
              v
          IN_QUEUE  ←── set by setRequestItemInQueueStatus()
              |         and setRequestItemInQueueStatusAsPerUpdateId()
              v
        IMPORTED  → COMPLETED
              |
              +----> IGNORED   (orphaned, or deleted job entity)
              +----> CANCELLED
              +----> SCANNED
```

- `IN_QUEUE` marks that the row has been enqueued in a Drupal queue for processing.
- `TO_PROCESS` marks rows the reconciler picked up but has not yet queued.
- `IGNORED` is used to mark orphans (see § "Orphan reconciliation" below).
- `IMPORTED` / `COMPLETED` are terminal for a successful import.

## CAPI-side lifecycle (`statuscode` column)

Values come from Lionbridge status updates (`StatusUpdate`, `StatusUpdateCtt` in the Swagger client). The module treats `CREATED` and `SENDING` as *not-yet-in-translation* — see the "in-flight" check at ~L1373 of `CapiDataProcessor` where those two are filtered together with the `file_upload_status` subset.

`SENT_TO_PROVIDER`, `IN_TRANSLATION`, `REVIEW_TRANSLATION`, `TRANSLATION_APPROVED` are the mid-life states. `CANCELLED` is terminal.

## Upload sub-state (`file_upload_status`) — added by update 9103

Applies only to rows in `statuscode IN ('CREATED', 'SENDING')`. Introduced to eliminate a race condition where two containers both attempted the upload. Also see `file_upload_attempts` (retry counter).

Only these column transitions have been confirmed:
- initial write: usually `PENDING`, or NULL for legacy rows.
- takeover: `PENDING → UPLOADING` guarded by an atomic UPDATE.
- success: `UPLOADING → UPLOADED`.
- failure: `UPLOADING → FAILED`, `file_upload_attempts` incremented.

Do not add new values without an update hook + config-parity across every query that filters on this column.

## Send flow (SEND category)

Roughly (verify by reading the exact `Services/CreateConnectorJob.php` + `Services/JobUploadManagerService.php` + the queue workers):

1. TMGMT plugin `ContentApiTranslator::requestTranslation` → `create_job` service.
2. `CreateConnectorJob` enqueues into `generate_file_for_translation_to_capi` (constant `QUEUE_NAME_GENERATE_FILES` in `QueueOperations`).
3. `GenerateFileFromQueue` worker calls `ExportJobFiles` → XLIFF file(s), plus optional ZIP bundling (`ZIP_JOB_PATH`, `ZIP_EXTENSION`, `PUBLIC_SENT_FILE_PATH` constants in `QueueOperations`).
4. File info is recorded in the `file_data` column of `tmgmt_capi_request_processor` (update 9106) — never in `\Drupal::state()` for cross-container safety.
5. Rows enter `statuscode = CREATED`, `file_upload_status = PENDING`.
6. `SendFilesToCapiFromQueue` picks them up, atomic-transitions to `UPLOADING`, calls CAPI via `CapiDataProcessor` / Swagger client. `HandleThrottling` handles rate limits.
7. On success: `statuscode → SENDING → SENT_TO_PROVIDER`, `file_upload_status → UPLOADED`, `status → IN_QUEUE`.

*ZIP path vs non-ZIP path:* both exist. The ZIP path bundles multiple items into one CAPI request; the non-ZIP path sends one file per request. **Every SEND-category change must be tested against both.**

## Import flow (IMPORT category)

1. Poller / status update triggers `ImportJob::execute` (from `ImportTranslatedJobsFromQueue`, or `ImportJobsManuallyFromQueue` for manual triggers).
2. `CapiDataProcessor::add_status_data` (see index `add_status_data`) inserts/updates a row per CAPI request.
3. When CAPI reports `TRANSLATION_APPROVED`, `CapiDataProcessor` marks matching rows `status = TO_PROCESS`.
4. `TO_PROCESS` → `IN_QUEUE` when enqueued.
5. Worker pulls the translation, writes back into TMGMT job items, transitions to `IMPORTED` then `COMPLETED`.

## Redelivery detection (update 9107 / v9.4.3)

If a TMGMT job is already `FINISHED` in Drupal but CAPI sends a further status update, the module resets that job to `ACTIVE`. This is guarded by the `idx_redelivery_check` composite index. See `CapiDataProcessor` L472–L595. **Do not remove this check.** Symptom of removal: silent loss of late-arriving translation updates.

## Orphan reconciliation

When the source TMGMT job entity is deleted but rows remain:
- `CapiDataProcessor` L377 marks orphaned `CREATED` rows to stop future matching.
- `QueueOperations` provides a helper that marks `TO_PROCESS`/`IN_QUEUE` rows for an orphan job as `IGNORED` (~L1231).

## Cross-environment safeguard (`source_site`, update 9109)

Every row is tagged with the current site identifier at insert. Every reconciliation query joins on `source_site = current OR source_site IS NULL` so that a **DB restored into a different environment** does not process another site's in-flight jobs. NULL is tolerated for pre-9109 legacy rows.

## Event flow (Auto-Bundle S21)

`AutoBundleAccumulatorSubscriber` listens on TMGMT job-item creation events for **continuous jobs on a `contentapi` translator with Auto-Bundle enabled**. It:
1. Composes `group_key` via `AutoBundleGroupKeyBuilder` (always-on dims: source lang, target lang, translator, continuous flag; optional: content type, priority tier).
2. Inserts into `tmgmt_contentapi_bundle_queue` (dedup on `uq_dedup` unique key).
3. Calls `AutoBundleTriggerEvaluator` — evaluates word/item/wait thresholds against caps.
4. If a trigger fires, `AutoBundleFlusher` (guarded by `@lock`) creates the actual TMGMT translation job and clears the queue rows for that group_key.

## Workflow gate (S20)

`WorkflowGateSubscriber` refuses to create a translation job when the source entity is in a non-published workflow state. Behavior is opt-in per translator setting.
