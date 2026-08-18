# 10 — Review checklist

Use this as the last step before you emit the output template in section 4 of the chatmode. If any box is unchecked, either fix the diff or explicitly justify it in "Unknowns / to verify".

## Scope discipline
- [ ] Change is limited to the requested task. No incidental refactor, rename, or docblock churn.
- [ ] Only files in `web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/` were modified (or a legitimate config/test file).
- [ ] No edits under `web/modules/contrib/tmgmt/**` (TMGMT core).
- [ ] No edits under `web/modules/contrib/lionbridge_translation_provider/**` (upstream mirror).
- [ ] No edits under `src/Swagger/Client/**` unless generator was rerun.

## Root cause
- [ ] Root cause is stated in the summary. Symptom-only fixes are called out as temporary.
- [ ] The fix touches the mechanism, not the log line.

## Status / DB safety (see 03, 04, 07)
- [ ] `status` vs `statuscode` used correctly (not conflated).
- [ ] Every new query filtering in-flight rows includes the `source_site` predicate.
- [ ] Every new claim on `CREATED`/`SENDING` rows uses atomic UPDATE on `file_upload_status`.
- [ ] No literal status string introduced without a matching constant on `CapiDataProcessor`.
- [ ] No new `\Drupal::state()` for cross-container file/job memory.

## Schema (if applicable)
- [ ] New column / index / table has a new `tmgmt_contentapi_update_9XXX()` hook.
- [ ] Corresponding schema function (`tmgmt_contentapi_defined_schema` / `_response_defined_schema` / `_bundle_queue_schema` / `_tmgmt_contentapi_table_indexes`) was updated for fresh installs.
- [ ] Update hook uses `fieldExists` / `indexExists` / `tableExists` guards.
- [ ] Historical hooks were not modified.

## Flows explicitly considered
- [ ] ZIP path — impacted / unchanged / N/A stated.
- [ ] Non-ZIP path — same.
- [ ] Single-item — same.
- [ ] Bulk — same.
- [ ] Multi-container — same.
- [ ] Continuous vs one-off job — same.

## Concurrency
- [ ] Any bundle-flush path holds `@lock` scoped to `group_key`.
- [ ] `AutoBundleGroupKeyBuilder` output remains deterministic.
- [ ] Continuous re-queue changes are covered by `ContinuousReQueueSuppressor` where applicable.

## Historical regressions (04-known-issues-safeguards.md)
- [ ] S1 (multi-container file memory) preserved.
- [ ] S2 (upload race) preserved.
- [ ] S3 (cross-env source_site) preserved.
- [ ] S4 (redelivery detection) preserved.
- [ ] S5 (orphan marking) preserved.
- [ ] S6 (stuck SENDING/IN_QUEUE recovery methods) not weakened.
- [ ] S7 (PHP 8.4) compatibility not regressed.
- [ ] S9 (Auto-Bundle correctness) preserved.
- [ ] S10 (continuous re-queue loops) preserved.

## PHP / Drupal compatibility
- [ ] Runs on PHP 8.1, 8.2, 8.3, 8.4.
- [ ] Runs on Drupal 9, 10, 11.
- [ ] No `#[\ReturnTypeWillChange]` introduced.
- [ ] No bare `TypedParam $x = NULL` (must be `?TypedParam $x = NULL`).
- [ ] All new/changed methods have return type declarations.
- [ ] External-string boundaries use `?? ''` before `explode/substr/implode/strlen`.

## Tests
- [ ] Unit test added or updated when the change is testable in isolation.
- [ ] If not unit-testable, manual repro procedure documented per [08-testing-guide.md](08-testing-guide.md).
- [ ] No new kernel/functional/browser test scaffolding without explicit approval.

## Config / UX
- [ ] New settings have a schema entry AND a form field AND a default.
- [ ] Logs use placeholders, not string concatenation. No secrets in logs.
- [ ] Operator report page (`Reports → TMGMT_CONTENTAPI`) still surfaces relevant errors.

## Documentation
- [ ] If a safeguard was added → row appended to [04-known-issues-safeguards.md](04-known-issues-safeguards.md) and [07-do-not-break.md](07-do-not-break.md).
- [ ] If schema changed → row appended to [05-release-history.md](05-release-history.md) with the update hook number.
- [ ] Version bumped in `info.yml` when releasing.
