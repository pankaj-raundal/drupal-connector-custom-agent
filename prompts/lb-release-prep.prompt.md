---
mode: 'lionbridge-maintainer'
description: 'Prepare a Lionbridge Content API module release (version bump, update hooks, docs, regression check).'
---

# Prepare a Lionbridge Content API release

## Inputs I will provide

- Target version (e.g. `9.4.8`).
- List of merged PRs / stories since previous release, if available.

## Procedure

1. Load [05-release-history.md](../lionbridge-agent/knowledge/05-release-history.md), [04-known-issues-safeguards.md](../lionbridge-agent/knowledge/04-known-issues-safeguards.md), [06-development-rules.md](../lionbridge-agent/knowledge/06-development-rules.md), [07-do-not-break.md](../lionbridge-agent/knowledge/07-do-not-break.md).
2. Determine the delta since the last recorded version:
   - Any new column / index / table? → confirm a new `tmgmt_contentapi_update_9XXX()` exists AND is idempotent.
   - Any change to translator settings? → confirm schema entry in `config/schema/tmgmt_contentapi.translator.schema.yml`.
   - Any new safeguard? → row must appear in [04-known-issues-safeguards.md](../lionbridge-agent/knowledge/04-known-issues-safeguards.md) and [07-do-not-break.md](../lionbridge-agent/knowledge/07-do-not-break.md).
3. Version bump:
   - Update `version:` in `tmgmt_contentapi/tmgmt_contentapi.info.yml`.
   - Add a row to [05-release-history.md](../lionbridge-agent/knowledge/05-release-history.md).
4. Run the unit test suite (see [08-testing-guide.md](../lionbridge-agent/knowledge/08-testing-guide.md)). Report pass/fail.
5. Regression sweep — for each of S1..S10 in [04-known-issues-safeguards.md](../lionbridge-agent/knowledge/04-known-issues-safeguards.md), confirm the safeguard is still present via a targeted `grep_search`:
   - S1 → `file_data` column referenced in `QueueOperations`.
   - S2 → atomic UPDATE on `file_upload_status = 'PENDING'` in `JobUploadManagerService`.
   - S3 → `source_site` predicate in reconciliation queries in `CapiDataProcessor` / `QueueOperations`.
   - S4 → redelivery-detection query in `CapiDataProcessor` L472–L595.
   - S5 → orphan-marking paths in `CapiDataProcessor` (~L377) and `QueueOperations` (~L1231).
   - S6 → recovery methods present on `QueueOperations`.
   - S7 → no `#[\ReturnTypeWillChange]`; no bare `Type $x = NULL`.
   - S9 → `@lock` acquisition in `AutoBundleFlusher`; `uq_dedup` unique key present on `tmgmt_contentapi_bundle_queue`.
   - S10 → `ContinuousReQueueSuppressor` still wired.
6. Confirm no changes leaked into `web/modules/contrib/lionbridge_translation_provider/**` or `web/modules/contrib/tmgmt/**`.
7. Emit a release-note summary with:
   - Version and date.
   - New features (map to `@dai-story` where applicable).
   - Bug fixes (map to symptoms).
   - Schema changes (list update hooks and idempotency guard).
   - PHP/Drupal compatibility statement.
   - Upgrade instructions (`drush updatedb`, `drush cr`).

## Do not

- Do not tag / push. Human tags the release.
- Do not delete or rewrite historical update hooks.
- Do not modify hook `9104` — that number is a documented gap.
