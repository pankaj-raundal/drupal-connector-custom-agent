# Lionbridge Maintainer Agent — Knowledge Router

This directory is the agent's structured knowledge base. **The chatmode file does NOT auto-load these.** The agent loads them on demand per section 1 of [lionbridge-maintainer.chatmode.md](../chatmodes/lionbridge-maintainer.chatmode.md).

## Files

| File | Purpose | When to load |
|---|---|---|
| [01-architecture.md](knowledge/01-architecture.md) | Module layout, services, plugins | Any structural / "where is X" question |
| [02-workflow-state-machine.md](knowledge/02-workflow-state-machine.md) | `status` + `statuscode` lifecycles, queue workers, event flow | Any SEND / IMPORT / STATE work |
| [03-database-schema.md](knowledge/03-database-schema.md) | All three tables, columns, indexes | Any DB / schema / update-hook work |
| [04-known-issues-safeguards.md](knowledge/04-known-issues-safeguards.md) | Historical bugs, root causes, safeguards | Any bug fix, review, refactor |
| [05-release-history.md](knowledge/05-release-history.md) | Update hooks and release notes | RELEASE, SCHEMA, and regression triage |
| [06-development-rules.md](knowledge/06-development-rules.md) | Coding conventions, tests, PHP/Drupal support matrix | Any change |
| [07-do-not-break.md](knowledge/07-do-not-break.md) | Hard invariants — check before every edit | Every change (small file) |
| [08-testing-guide.md](knowledge/08-testing-guide.md) | How to run and write unit tests | Adding/changing tests |
| [09-file-map.md](knowledge/09-file-map.md) | Compact index of every source file with 1-line purpose | Locating a subsystem |
| [10-review-checklist.md](knowledge/10-review-checklist.md) | The reviewer's checklist | REVIEW step of every task |

## Category → what to load (progressive-load router)

Load **only** the rows matching the classified category. Do not preload others.

| Category | Knowledge files | Primary source files |
|---|---|---|
| **SEND** (export/upload) | 02, 03, 04, 07 | `Services/CreateConnectorJob.php`, `Services/JobUploadManagerService.php`, `Services/ExportJobFiles.php`, `Plugin/QueueWorker/GenerateFileFromQueue.php`, `Plugin/QueueWorker/SendFilesToCapiFromQueue.php`, `Plugin/QueueWorker/ExportTranslationJobsToCapiFromQueue.php`, `Services/HandleThrottling.php` |
| **IMPORT** | 02, 03, 04, 07 | `Services/ImportJob.php`, `Services/CapiDataProcessor.php`, `Plugin/QueueWorker/ImportTranslatedJobsFromQueue.php`, `Plugin/QueueWorker/ImportJobsManuallyFromQueue.php` |
| **STATE** (statuscode/status/orphan/redelivery/source_site) | 02, 03, 04, 07 | `Services/CapiDataProcessor.php`, `Services/QueueOperations.php` |
| **AUTOBUNDLE** (S21+) | 03, 06, 07 | `Services/AutoBundleGroupKeyBuilder.php`, `Services/AutoBundleTriggerEvaluator.php`, `Services/AutoBundleFlusher.php`, `Services/AutoBundleGroupKeyHumanizer.php`, `Services/AutoBundleStatusBuilder.php`, `EventSubscriber/AutoBundleAccumulatorSubscriber.php`, `Form/AutoBundleForm.php`, `Controller/AutoBundleStatusController.php`, `Services/PriorityFieldDiscovery.php` |
| **CONTINUOUS** | 02, 04, 07 | `Services/ContinuousJobService.php`, `EventSubscriber/ContinuousReQueueSubscriber.php`, `ContinuousReQueueSuppressor.php`, `FormHelper/ContinuousJobFormHelper.php`, `Controller/ContinuousSubmissionsController.php` |
| **SCHEMA** (update hooks / columns / indexes) | 03, 05, 07 | `tmgmt_contentapi.install`, `config/schema/tmgmt_contentapi.translator.schema.yml` |
| **UI** (forms, blocks, views) | 06 | `Form/AutoBundleForm.php`, `ContentApiTranslatorUI.php`, `Plugin/Block/QueueStatusBlock.php`, `Plugin/views/**`, `EventSubscriber/WorkflowGateSubscriber.php`, `FormHelper/**` |
| **COMPAT** (PHP 8.4 / Drupal 11 / return types) | 06, 07 | Whichever file emits the deprecation |
| **REVIEW** | 04, 07, 10 | Whatever the diff touches |
| **RELEASE** | 05, 06, 07 | `tmgmt_contentapi.info.yml`, `tmgmt_contentapi.install`, CHANGELOG |

## Rules for editing this knowledge base

- Keep each file **short**. If a file exceeds ~300 lines, split it.
- Every fact must trace back to code. When in doubt, mark `UNKNOWN — verify`.
- Never mirror large source blocks verbatim; describe them, name the file+function, and link.
- On every module release, update **only the deltas** in 04, 05, 07.
