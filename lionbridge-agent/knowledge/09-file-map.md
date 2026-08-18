# 09 — File map (quick index)

Compact one-line description of every non-generated file in `tmgmt_contentapi/`. Load this file when you know *what* you want but not *where* it lives.

Path prefix omitted; all relative to `web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/`.

## Root

- `tmgmt_contentapi.info.yml` — module metadata, version, core range, `tmgmt` dep.
- `tmgmt_contentapi.install` — 3 schemas + all `hook_update_9XXX` and helper functions.
- `tmgmt_contentapi.module` — hook implementations (theme, form_alter, cron, views, entity).
- `tmgmt_contentapi.services.yml` — ~20 services + 3 subscribers + 1 decorator.
- `tmgmt_contentapi.routing.yml` — queue-process endpoint, continuous-submissions, Auto-Bundle form + status refresh.
- `tmgmt_contentapi.links.task.yml` — local tasks (Edit, Submissions, Auto-Bundle).
- `tmgmt_contentapi.permissions.yml` — permission definitions.
- `tmgmt_contentapi.libraries.yml` — JS/CSS libraries.
- `config/schema/tmgmt_contentapi.translator.schema.yml` — translator settings incl. `auto_bundle.*`.
- `README.md` — user-facing setup + Auto-Bundle prerequisites.

## `src/Plugin/tmgmt/Translator/`
- `ContentApiTranslator.php` — TMGMT plugin `id = "contentapi"`, `ContinuousTranslatorInterface`, thin dispatch to `create_job`.

## `src/Plugin/tmgmt_contentapi/Format/`
- `Xliff.php` — XLIFF export/import format plugin.

## `src/Plugin/QueueWorker/`
- `GenerateFileFromQueue.php` — generate translation file for one job.
- `SendFilesToCapiFromQueue.php` — upload generated file to CAPI.
- `ExportTranslationJobsToCapiFromQueue.php` — orchestrate export.
- `ImportTranslatedJobsFromQueue.php` — pull translated content back.
- `ImportJobsManuallyFromQueue.php` — manual-import variant.
- `MigrateJobsFromQueue.php` — one-time migration to new row structure.

## `src/Plugin/Block/`
- `QueueStatusBlock.php` — dashboard block showing queue counts.

## `src/Plugin/views/`
- `filter/LioxJobIdFilter.php`, `filter/LioxJobstatusFilter.php` — views filters.
- `field/TmgmtCapiItemsCount.php`, `field/JobStatusField.php`, `field/JobProvideridField.php`, `field/JobLioxidField.php` — views fields.

## `src/Services/` (see 01-architecture for grouped roles)

- `CapiDataProcessor.php` (1422 lines) — CAPI status ingestion, request-processor row lifecycle, redelivery, orphan marking.
- `QueueOperations.php` (1340 lines) — queue helpers, file-transfer manifest via `file_data`, cleanup + recovery methods.
- `JobUploadManagerService.php` (1310 lines) — atomic upload claim, retry, failure surfacing.
- `CreateConnectorJob.php` (1069 lines) — job submission orchestration.
- `ImportJob.php` (896 lines) — translation import back to TMGMT.
- `AutoBundleStatusBuilder.php` (746 lines) — S22 status-panel data.
- `ContinuousJobService.php` (549 lines) — continuous job helpers.
- `AnalysisCodeApi.php` (542 lines) — analysis-code API integration.
- `AutoBundleFlusher.php` (475 lines) — S21 flush-with-lock.
- `ExportJobFiles.php` (403 lines) — XLIFF file generation.
- `JobHelper.php` (361 lines) — filesystem/user/mime helpers.
- `AutoBundleGroupKeyBuilder.php` (225 lines) — S21 deterministic group key.
- `CapiDetails.php` (171 lines) — provider/CAPI meta.
- `PriorityFieldDiscovery.php` (170 lines) — S5 list-field lookup.
- `AutoBundleTriggerEvaluator.php` (163 lines) — S21 threshold logic.
- `AutoBundleGroupKeyHumanizer.php` (147 lines) — S22 human labels.
- `HandleThrottling.php` (116 lines) — 429/backoff handling.
- `NullSafeLinkGenerator.php` (51 lines) — decorator for `link_generator`.

## `src/EventSubscriber/`
- `AutoBundleAccumulatorSubscriber.php` — S21, writes to `tmgmt_contentapi_bundle_queue`.
- `ContinuousReQueueSubscriber.php` — re-queues continuous items.
- `WorkflowGateSubscriber.php` — S20, gates job creation by workflow state.

## `src/Form/`
- `AutoBundleForm.php` — Auto-Bundle tab; grouping dims, triggers, caps, submission mode.

## `src/FormHelper/`
- `ContinuousJobFormHelper.php`
- `ContentApiAnalysisCodeFormHelper.php`

## `src/Controller/`
- `QueueProcessController.php` — POST `/tmgmt-contentapi/queue-process-background/{queue_name}/{batch_size}`.
- `ContinuousSubmissionsController.php` — `/admin/tmgmt/jobs/{tmgmt_job}/submissions`.
- `AutoBundleStatusController.php` — `/admin/tmgmt/translators/manage/{tmgmt_translator}/auto-bundle/status` (AJAX).

## `src/` (loose)
- `ContentApiTranslatorUI.php` — TMGMT translator UI class (settings, checkout).
- `ContinuousReQueueSuppressor.php` — scoped suppression flag for the continuous re-queue subscriber.
- `RecursiveDOMIterator.php` — helper for XLIFF DOM walking.
- `Annotation/`, `Format/` — plugin annotation / format manager scaffolding.

## `src/Swagger/Client/` — **generated, do not hand-edit**
- `Configuration.php`, `HeaderSelector.php`, `ObjectSerializer.php` — client infra.
- `Api/JobApi.php`, `Api/ProviderApi.php`, `Api/TokenApi.php`, `Api/RequestApi.php`, `Api/FileApi.php`, `Api/SourceFileApi.php`, `Api/StatusUpdateApi.php`, `Api/TranslationMemoryApi.php`, `Api/TranslationContentApi.php`, `Api/SupportAssetApi.php` — CAPI endpoints.
- `Model/*.php` — request/response models.

## `tests/src/Unit/`
- `EventSubscriber/WorkflowGateSubscriberTest.php`
- `Services/AnalysisCodeApiTest.php` (+ `.md` notes)
- `Services/AutoBundleGroupKeyBuilderTest.php`
- `Form/AutoBundleFormTest.php`

## Templates / assets
- `templates/`, `css/`, `js/`, `icons/` — S22 panel + Auto-Bundle UI assets, translator logo.
