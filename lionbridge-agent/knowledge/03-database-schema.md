# 03 — Database schema

The module owns **three tables**. Definitions live in [tmgmt_contentapi.install](../../../web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/tmgmt_contentapi.install) as `tmgmt_contentapi_defined_schema()`, `tmgmt_contentapi_response_defined_schema()`, `tmgmt_contentapi_bundle_queue_schema()`.

## Table 1 — `tmgmt_capi_request_processor`

The workhorse table. One row per CAPI request tied back to a TMGMT job/item.

| Column | Type | Notes |
|---|---|---|
| `rid` | serial (PK) | Unique row id |
| `tjid` | int, default 0 | TMGMT job id |
| `tjiid` | int, default 0 | TMGMT job item id |
| `updateid` | varchar(255) | Correlates a status-update batch. Default `''`. |
| `jobid` | varchar(100), **NOT NULL** | CAPI job id (aka `capi_job_id`) |
| `providerid` | varchar(255) | CAPI provider id |
| `requestid` | varchar(255), **NOT NULL** | CAPI request id |
| `statuscode` | varchar(255) | CAPI-side state (see [02](02-workflow-state-machine.md)) |
| `haserror` | tinyint | 0/1 |
| `errormessage` | varchar(255) | |
| `status` | varchar(50) | Row-workflow (see [02](02-workflow-state-machine.md)) |
| `updatedtime` | datetime(6) | CAPI-reported timestamp |
| `lastupdated` | timestamp | Drupal-side last-modified |
| `file_upload_status` | varchar(50), nullable | update 9103. Only for `CREATED`/`SENDING` rows. |
| `file_upload_attempts` | tinyint, nullable | update 9103. Retry counter. |
| `capi_response` | text, nullable | update 9103. Cached API response payload. |
| `file_data` | text (big), nullable | **update 9106.** Serialized `{file_id, file_uri, filename, item_id}` — replaces `\Drupal::state()` for multi-container consistency. |
| `source_site` | varchar(255), nullable | **update 9109.** Domain of the site that inserted the row. Prevents cross-env processing after DB restore. NULL = legacy row = treat as current site. |

### Indexes (from `_tmgmt_contentapi_table_indexes()`)

Do not remove or rename these; they back specific queries whose names give away their purpose. Adding a new query? Add a matching index in a new update hook.

- `rid`
- `setRequestProgressStatus` (`status`, `requestid`)
- `add_status_data` (`jobid`, `requestid`, `statuscode`)
- `getAllReadyRequestIdToImport` (`jobid`, `requestid`, `statuscode`)
- `getAllReadyRequestIdToImport_2` (`status`, `tjid`)
- `setRequestProgressStatusUsingItemId` (`status`, `tjiid`)
- `status`, `jobid`
- `updatePreviousRequestStatusToNew` (`status`, `tjiid`, `tjid`, `requestid`)
- `getCapiProcessorDetailsBasisOftjid` (`statuscode`, `tjiid`)
- `setRequestItemInQueueStatusAsPerUpdateId` (`status`, `updateid`)
- `deleteIgnoredRecords` (`lastupdated`, `status`)
- `deleteProcessorRecords` (`tjid`, `jobid`)
- `getActiveUploadCount` (`file_upload_status`, `statuscode`, `tjid`)
- `composite_tjid_tjiid_statuscode`
- `composite_tjid_statuscode_file_upload`
- `composite_status_providerid_tjid`
- `composite_tjid_jobid_statuscode`
- `composite_statuscode_requestid_jobid`
- `tjiid_statuscode`
- `idx_redelivery_check` (`status`, `jobid`, `statuscode`, `requestid`, `updatedtime`, `tjid`) — **update 9107**
- `idx_jobid_source_site` (`jobid`, `source_site`) — **update 9109**
- `idx_job_overview_sort`, `idx_file_data_lookup`, `idx_status_job_request_group`, `idx_tjid_tjiid_sort` — **update 9108**

## Table 2 — `tmgmt_capi_response`

Stores per-item API responses.

| Column | Type | Notes |
|---|---|---|
| `respid` | serial (PK) | |
| `jobkey` | varchar(100), NOT NULL | Batch key |
| `item_id` | varchar(50), NOT NULL | Item within the batch |
| `apiresponse` | text, NOT NULL | Raw response |
| `created` | int | Insert timestamp |

Index: `idx_jobkey_item` on `(jobkey, item_id)` — **update 9108**.

## Table 3 — `tmgmt_contentapi_bundle_queue` (update 9110, S21)

Holds continuous-job items waiting for auto-bundle trigger evaluation.

| Column | Type | Notes |
|---|---|---|
| `id` | serial (PK) | |
| `translator_id` | varchar(128), NOT NULL | TMGMT translator machine name |
| `group_key` | varchar(255), NOT NULL | Deterministic key from `AutoBundleGroupKeyBuilder` |
| `entity_type` / `entity_id` | varchar(128) / int | Source entity |
| `langcode_source`, `langcode_target` | varchar(12) | |
| `word_count` | int | Snapshot at accumulation time |
| `priority` | varchar(64), nullable | Value from configured priority field (NULL if not grouped by priority) |
| `plugin` | varchar(128) | TMGMT source plugin id, default `content` |
| `continuous_job_id` | int, NOT NULL | Parent continuous job |
| `created` | int, NOT NULL | Timestamp |

Unique key: `uq_dedup` on `(translator_id, group_key, entity_type, entity_id, langcode_target)` — enforces dedup.
Indexes: `idx_translator_group`, `idx_translator_created`, `idx_group_created`.

## Rules for schema changes

1. **New column** → add via new `tmgmt_contentapi_update_9XXX()` hook AND update `tmgmt_contentapi_defined_schema()` / relevant schema function so fresh installs match.
2. **New index** → same pattern; add to `_tmgmt_contentapi_table_indexes()` and the update hook.
3. **Never edit historical update hooks.** They may have already run in production.
4. **Never assume order of updates.** Each update hook checks existence first (`fieldExists`, `indexExists`, `tableExists`) — follow that convention.
5. **Backfill strategy:** the codebase's convention is to *leave existing rows NULL* and treat NULL as "legacy / same environment / not tracked" (see `source_site`, `file_upload_status` comments in the install file). Follow this pattern unless you have a compelling reason to backfill.
6. **Uniqueness enforcement:** on adding new dedup logic, prefer a real unique key like `uq_dedup` on the bundle table, not application-level checks alone.
