# 04 — Known issues & safeguards

Every entry here fixed a real production incident. **Do not weaken any safeguard without an explicit user request.** Extend, don't replace.

## S1 — Multi-container file memory (update 9106, v9.4.x)

- **Symptom:** Jobs stuck in "pending"; files failed to upload; intermittent success on multi-container hosting (Pantheon, Docker, Kubernetes).
- **Root cause:** File info (`file_id`, `file_uri`, `filename`, `item_id`) was stored in `\Drupal::state()`, which is process-local per container. Container A wrote it; container B read empty.
- **Fix:** Added `file_data` (text/big) column on `tmgmt_capi_request_processor`. All file-transfer manifests now flow through the DB.
- **Do not:** reintroduce `\Drupal::state()` for cross-request file/job memory.
- **Reference doc:** [`MULTI_CONTAINER_FILE_HANDLING_GUIDE.md`](../../../MULTI_CONTAINER_FILE_HANDLING_GUIDE.md).

## S2 — File-upload race condition (update 9103)

- **Symptom:** Duplicate uploads, `SENDING → FAILED` even when CAPI accepted.
- **Root cause:** Two workers claimed the same `CREATED` row simultaneously.
- **Fix:** `file_upload_status` column with an atomic `UPDATE … WHERE file_upload_status = 'PENDING'` guard; `file_upload_attempts` for retry accounting; `capi_response` for auditing.
- **Do not:** skip the atomic UPDATE guard when adding a new claim path; do not use plain SELECT-then-UPDATE.

## S3 — Cross-environment processing after DB restore (update 9109)

- **Symptom:** After restoring prod DB to a lower env (test/dev), that lower env would call CAPI and *modify real production jobs* because it saw in-flight rows.
- **Root cause:** No environment marker on rows.
- **Fix:** `source_site` column; reconciliation queries add `source_site = current OR source_site IS NULL`. NULL means legacy — treat as current env.
- **Do not:** query in-flight rows without the `source_site` predicate; do not backfill legacy NULLs.

## S4 — Late-arriving translations lost (update 9107, v9.4.3)

- **Symptom:** A job marked FINISHED never received the last status update; the translation silently disappeared.
- **Root cause:** No code path re-opened a FINISHED job when CAPI sent a further status update.
- **Fix:** Redelivery detection query (backed by `idx_redelivery_check`) resets FINISHED jobs to ACTIVE when a new status update arrives. See `CapiDataProcessor` L472–L595.
- **Do not:** remove or gate the redelivery check without an approved regression test.

## S5 — Orphan rows after job entity deletion

- **Symptom:** Zombie rows in `tmgmt_capi_request_processor` for TMGMT jobs that no longer exist; workers churning on rows that will never complete.
- **Fix:** Two-part.
  - `CapiDataProcessor` marks orphaned `CREATED` rows to stop matching (~L377).
  - `QueueOperations` marks `TO_PROCESS`/`IN_QUEUE` rows for an orphan `jobid` as `IGNORED` (~L1231).
- **Do not:** skip the orphan check when adding a new consumer of these rows.

## S6 — SENDING / IN_QUEUE getting stuck (historical pattern)

- **Pattern:** Rows that entered `statuscode = SENDING` or `status = IN_QUEUE` never advanced because a worker crashed mid-flight.
- **Ongoing safeguards:**
  - Atomic PENDING→UPLOADING guard prevents double-claim.
  - `file_upload_attempts` bounds retries; `FAILED` is terminal after N tries.
  - `getActiveUploadCount` index supports operator dashboards.
  - `getFailedUploadItems`, `requeueFailedItems`, `cleanupPartialJobFailure`, `getCompleteCleanupFailures`, `resetCompleteCleanupFailure`, `propagateSuccessToWaitingItems` methods on `QueueOperations` — the operational recovery toolbox.
- **When fixing a new stuck-row scenario:** first identify which of these safeguards *should* have caught it, and why it didn't. That's the root cause.

## S7 — PHP 8.4 deprecations (v9.4.5)

- Fixed 3 real bugs (`+=` used for string concat, undefined `$msg`, `implode(NULL)`), 5 implicit-nullable params, 10 null-to-internal-fn call sites, ~53 `#[\ReturnTypeWillChange]` methods now have proper return types.
- Reference: [`PHP_8.4_COMPATIBILITY_SUMMARY.md`](../../../PHP_8.4_COMPATIBILITY_SUMMARY.md).
- **Do not:** reintroduce `#[\ReturnTypeWillChange]` or bare `= NULL` typed params.

## S8 — Memory / timeout (v9.4.x)

- Reference doc: [`MEMORY_TIMEOUT_FIX.md`](../../../MEMORY_TIMEOUT_FIX.md) *(UNKNOWN — file is 0 bytes at time of writing; verify contents before citing details).* Behavior likely relates to batching / composite indexes added in updates 9105 and 9108.

## S9 — Auto-Bundle correctness (S21, v9.4.x)

- **Dedup:** `uq_dedup` unique key on `tmgmt_contentapi_bundle_queue` prevents double-accumulation of the same entity/target.
- **Concurrency:** `AutoBundleFlusher` uses `@lock` service to prevent two flushers colliding on the same `group_key`.
- **Determinism:** `AutoBundleGroupKeyBuilder` composes a canonical, ordered key. Never allow non-determinism (map iteration order, unsorted arrays) in the key.
- **Caps:** `cap_max_items`, `cap_max_words`, `cap_min_items` are enforced by `AutoBundleTriggerEvaluator`. `cap_wait_overrides_min_items` lets wait-time triggers bypass min-items.

## S10 — Continuous re-queue loops

- **Suppressor pattern:** `ContinuousReQueueSuppressor` is a stateful service used to temporarily disable `ContinuousReQueueSubscriber` inside code paths that would otherwise re-enqueue during their own processing. If you add a code path that mutates continuous-job items, check whether it needs to enter/exit the suppressor.

## Known integrity patterns — check before proposing fixes

- **`updateid` semantics:** correlates a status-update batch. Multiple rows can share it. Queries against `updateid` should include a `status` filter (see index `setRequestItemInQueueStatusAsPerUpdateId`).
- **`tjiid` and `tjid` may be 0** (schema default). Some rows are job-level not item-level. Filter accordingly.
- **`updatedtime` is CAPI-side; `lastupdated` is Drupal-side.** Do not swap them.
