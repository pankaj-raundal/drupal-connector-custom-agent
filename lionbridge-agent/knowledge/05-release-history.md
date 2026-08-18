# 05 — Release history

Sources: [tmgmt_contentapi.info.yml](../../../web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/tmgmt_contentapi.info.yml), [tmgmt_contentapi.install](../../../web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/tmgmt_contentapi.install), and the top-level docs.

The module has no dedicated CHANGELOG file; release deltas must be reconstructed from update hooks + doc files. Keep this file in sync on each release.

## Current version — 9.4.7

## Update hooks (`hook_update_N` history)

| Hook | Purpose | Feature landed |
|---|---|---|
| `9100` | Create `tmgmt_capi_request_processor` | Initial request-tracking table |
| `9101` | Enqueue existing jobs into `migrate_jobs_to_new_structure_queue` | Migration to new structure |
| `9102` | Create `tmgmt_capi_response` | Response cache table |
| `9103` | Add `file_upload_status`, `file_upload_attempts`, `capi_response` columns | Race-condition fix (S2) |
| `9105` | Backfill all indexes from `_tmgmt_contentapi_table_indexes()` | Index consolidation |
| `9106` | Add `file_data` column | Multi-container file handling (S1) |
| `9107` | Add `idx_redelivery_check` index | Redelivery detection (S4) — v9.4.3 |
| `9108` | Add perf indexes on request + response tables | Query performance |
| `9109` | Add `source_site` column + `idx_jobid_source_site` | Cross-env safety (S3) |
| `9110` | Create `tmgmt_contentapi_bundle_queue` | Auto-Bundle runtime (S21) |

*Hook 9104 is absent — either never landed or renumbered. Do not fill the gap.*

## Version → known highlights (reconstructed)

- **9.4.3** — Redelivery detection index (`idx_redelivery_check`, update 9107).
- **9.4.4 → 9.4.5** — PHP 8.4 compatibility pass. Details in [`PHP_8.4_COMPATIBILITY_SUMMARY.md`](../../../PHP_8.4_COMPATIBILITY_SUMMARY.md).
- **9.4.x (unknown minor)** — Multi-container file handling: `file_data` column (update 9106).
- **9.4.x (unknown minor)** — Cross-env `source_site` (update 9109).
- **9.4.x (unknown minor)** — S20 Workflow gate subscriber.
- **9.4.x (unknown minor)** — S21 Auto-Bundle runtime + S22 status panel.

*Marked UNKNOWN where the exact minor is not documented in code. Verify against internal release notes before publishing a changelog entry.*

## Related upstream patches shipped in the workspace

These files at the repo root are patches applied to sibling modules or backports, not module releases per se:
- `lionbridge_translation_provider-3538329-02-import-action.patch`
- `lionbridge_translation_provider-3547889-01.patch`
- `liox_tmgmt_capi_request_processor_improvement.patch`

Do not silently reintroduce content from these patches into `tmgmt_contentapi/` without checking whether the module has since absorbed the fix.

## Release-prep checklist

1. Update `version:` in [tmgmt_contentapi.info.yml](../../../web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/tmgmt_contentapi.info.yml).
2. If schema changed, ensure a new `tmgmt_contentapi_update_9XXX()` hook exists and is idempotent (`fieldExists`, `indexExists`, `tableExists` guards).
3. Update this file (05) with the new row.
4. Update [04-known-issues-safeguards.md](04-known-issues-safeguards.md) if a new safeguard was added.
5. Update [07-do-not-break.md](07-do-not-break.md) if a new invariant was added.
6. Verify unit tests still pass: `phpunit -c web/core/phpunit.xml.dist tmgmt_contentapi/tests/src/Unit`.
7. Run the [10-review-checklist.md](10-review-checklist.md) once end-to-end.
8. Do NOT commit changes to `web/modules/contrib/lionbridge_translation_provider/` (upstream mirror).
