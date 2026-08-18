# 01 — Architecture

## Module identity

- Machine name: `tmgmt_contentapi` (submodule of `lionbridge_translation_provider`)
- Version: **9.4.7** ([info.yml](../../../web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/tmgmt_contentapi.info.yml))
- Core: `^9 || ^10 || ^11`
- Depends on: `tmgmt` (Translation Management Tool)
- Purpose: TMGMT translator plugin that submits content to Lionbridge Content API (CAPI) and imports translations back.

## Live path vs mirrors

- **Live / edit here:** `web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/`
- **Do not edit:** `web/modules/contrib/lionbridge_translation_provider/` (upstream mirror, read-only)
- **Do not edit:** `web/modules/contrib/tmgmt/` (TMGMT core)

## Top-level layout

```
tmgmt_contentapi/
├── tmgmt_contentapi.info.yml
├── tmgmt_contentapi.install          # 3 tables + update hooks 9100–9110
├── tmgmt_contentapi.module           # hooks (theme, form_alter, cron, entity, views)
├── tmgmt_contentapi.services.yml     # ~20 services
├── tmgmt_contentapi.routing.yml
├── tmgmt_contentapi.permissions.yml
├── tmgmt_contentapi.links.task.yml
├── tmgmt_contentapi.libraries.yml
├── config/schema/tmgmt_contentapi.translator.schema.yml   # incl. auto_bundle
├── src/
│   ├── Plugin/
│   │   ├── tmgmt/Translator/ContentApiTranslator.php     # the TMGMT plugin
│   │   ├── tmgmt_contentapi/Format/Xliff.php             # export format
│   │   ├── QueueWorker/                                   # 6 workers (see below)
│   │   ├── Block/QueueStatusBlock.php
│   │   └── views/{field,filter}/                          # 5 view plugins
│   ├── Services/                                          # 20 services (see below)
│   ├── EventSubscriber/                                   # 3 subscribers
│   ├── Form/AutoBundleForm.php                            # 1000+ lines, Auto-Bundle UI
│   ├── FormHelper/{ContinuousJobFormHelper, ContentApiAnalysisCodeFormHelper}.php
│   ├── Controller/                                        # 3 controllers
│   ├── ContentApiTranslatorUI.php                         # TranslatorPluginUiBase impl
│   ├── ContinuousReQueueSuppressor.php
│   ├── RecursiveDOMIterator.php
│   └── Swagger/Client/                                    # generated CAPI client (do not hand-edit)
└── tests/src/Unit/                                        # unit tests only
```

## Services (from `tmgmt_contentapi.services.yml`)

Grouped by role. Load each service's file only when your task category requires it.

### Job lifecycle
- `tmgmt_contentapi.create_job` → `Services/CreateConnectorJob.php` — orchestrates job submission to CAPI.
- `tmgmt_contentapi.export_job` → `Services/ExportJobFiles.php` — generates translation files (XLIFF).
- `tmgmt_contentapi.import_job` → `Services/ImportJob.php` — pulls translated content back.
- `tmgmt_contentapi.job_upload_manager` → `Services/JobUploadManagerService.php` — manages file uploads with retry/state (1300+ lines).
- `tmgmt_contentapi.job_helper` → `Services/JobHelper.php` — file/user/mime helpers.

### CAPI integration
- `tmgmt_contentapi.capi_data_processor` → `Services/CapiDataProcessor.php` — talks to `Swagger/Client`, manages `tmgmt_capi_request_processor` rows.
- `tmgmt_contentapi.capi_details` → `Services/CapiDetails.php`.
- `tmgmt_contentapi.handle_throttling` → `Services/HandleThrottling.php` — CAPI rate-limit handling.
- `tmgmt_contentapi.analysis_code_api` → `Services/AnalysisCodeApi.php` — analysis codes.

### Queue & state
- `tmgmt_contentapi.queue_operations` → `Services/QueueOperations.php` — queue helpers, file transfer manifest, cleanup (1300+ lines).

### Continuous jobs
- `tmgmt_contentapi.continuous_job` → `Services/ContinuousJobService.php`.
- `tmgmt_contentapi.continuous_requeue_subscriber` → `EventSubscriber/ContinuousReQueueSubscriber.php`.

### Auto-Bundle (S21 runtime engine + S22 status panel)
- `tmgmt_contentapi.auto_bundle_group_key_builder` → composes deterministic group key.
- `tmgmt_contentapi.auto_bundle_trigger_evaluator` → decides when to flush a bundle.
- `tmgmt_contentapi.auto_bundle_flusher` → creates the actual TMGMT job from a bundle (uses `@lock`).
- `tmgmt_contentapi.auto_bundle_accumulator_subscriber` → listens to TMGMT events, writes to `tmgmt_contentapi_bundle_queue`.
- `tmgmt_contentapi.auto_bundle_group_key_humanizer` → human labels for group keys.
- `tmgmt_contentapi.auto_bundle_status_builder` → cards for the right-side status panel.
- `tmgmt_contentapi.priority_field_discovery` → S5, finds list_string/list_integer fields on translatable bundles.

### Workflow gate
- `tmgmt_contentapi.workflow_gate_subscriber` → S20, blocks TMGMT job creation when source entity is in a non-published workflow state.

### Misc decorators
- `tmgmt_contentapi.null_safe_link_generator` → decorates `link_generator` to tolerate NULL URLs.
- `plugin.manager.tmgmt_contentapi.format` → format plugin manager.

## Queue workers (`src/Plugin/QueueWorker/`)

Small classes (~50 lines) that delegate to services.

| Worker | Queue name | Delegates to |
|---|---|---|
| `GenerateFileFromQueue` | `generate_file_for_translation_to_capi` | `export_job` / `create_job` |
| `SendFilesToCapiFromQueue` | (send-files queue) | `job_upload_manager` / `capi_data_processor` |
| `ExportTranslationJobsToCapiFromQueue` | (export-jobs queue) | `create_job` |
| `ImportTranslatedJobsFromQueue` | (import queue) | `import_job` |
| `ImportJobsManuallyFromQueue` | (manual import queue) | `import_job` |
| `MigrateJobsFromQueue` | `migrate_jobs_to_new_structure_queue` | `capi_data_processor` |

*Queue names sourced from constants in `QueueOperations` and `CapiDataProcessor`; confirm the exact machine names by `grep_search` before referencing in a fix.*

## Event subscribers

- `ContinuousReQueueSubscriber` — re-queues continuous job items on TMGMT events. Paired with `ContinuousReQueueSuppressor` service for scoped disabling.
- `WorkflowGateSubscriber` (S20) — blocks translation job creation based on source entity workflow state.
- `AutoBundleAccumulatorSubscriber` (S21) — accumulates items into `tmgmt_contentapi_bundle_queue`, then evaluates trigger and calls flusher.

## Controllers

- `QueueProcessController` — HTTP endpoint to process a queue in the background (CSRF-protected).
- `ContinuousSubmissionsController` — lists submissions per continuous job.
- `AutoBundleStatusController` — AJAX refresh endpoint for the Auto-Bundle status panel.

## Forms

- `AutoBundleForm` — Auto-Bundle configuration tab on the translator edit page. Fully implemented UI+validation for grouping dimensions, triggers, and safety caps.
- `ContentApiTranslatorUI` — TMGMT translator UI (settings form, checkout).

## Swagger client

`src/Swagger/Client/` is machine-generated. **Do not hand-edit.** If CAPI changes, regenerate; do not patch model classes for style. Bug-fix patches are acceptable only when the generator produces broken output — mark as such.
