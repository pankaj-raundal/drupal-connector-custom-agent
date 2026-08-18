# 06 — Development rules

## Support matrix (from info.yml + code)

- **Drupal core:** `^9 || ^10 || ^11`
- **PHP:** confirmed working on 8.1, 8.2, 8.3, 8.4 (see PHP 8.4 summary).
- **TMGMT:** required dependency (`tmgmt`); no version pin visible — assume "latest stable that supports the same core range".

Every change must remain compatible with **all three** Drupal majors and PHP 8.1–8.4 unless the user explicitly authorizes a bump plus a version increment.

## Coding conventions (as practiced in this module)

- **PSR-4** namespace `Drupal\tmgmt_contentapi\…` under `src/`.
- **Services** injected via `services.yml`; do not `\Drupal::service()` inside constructors of new services (existing code sometimes does; do not spread it further).
- **Constants** for statuses live on `CapiDataProcessor` (e.g. `IN_QUEUE`, `SENDING`, `TO_PROCESS`). Reuse them; never hardcode a status literal.
- **Queue names** and file-path constants live on `QueueOperations` (e.g. `QUEUE_NAME_GENERATE_FILES`, `ZIP_JOB_PATH`, `ZIP_EXTENSION`, `PUBLIC_SENT_FILE_PATH`).
- **Return types:** every new/changed public method must declare its return type (post PHP 8.4 pass). Do not use `#[\ReturnTypeWillChange]`.
- **Nullables:** every nullable typed param uses the `?T $x = NULL` form. No bare `T $x = NULL`.
- **Null-safety at boundaries:** external-source strings (HTTP headers, CAPI response fields, `getSourceNativeId()`, `getResponseBody()`) must be `?? ''` before being passed to `explode`, `substr`, `strlen`, `implode`, etc.
- **Logger channel:** `logger.factory` service, channel `tmgmt_contentapi`. Include an `@…` placeholder or a translation-aware log line — do not concatenate into the message.
- **Annotations for provenance (project convention):** contributions carry `@dai-story: SNN (title)` / `@dai-task: TNN (title)` comments. Keep this when editing story-tagged blocks; do not add new tags without a story id.

## What NOT to do

- No refactors, style changes, or "cleanup" that aren't part of the requested fix.
- No adding docblocks or `@param` blocks to methods you didn't functionally change.
- No new helper classes for one-time operations. Extend an existing service.
- No new event subscribers where a service method call would do.
- No new update hooks unless schema changed; don't renumber existing hooks.
- No modifications to `src/Swagger/Client/**` (generated code). If CAPI changes, regenerate.
- No modifications to files under `web/modules/contrib/lionbridge_translation_provider/` (upstream mirror) or `web/modules/contrib/tmgmt/` (TMGMT core).

## Testing rules

- **Unit tests only** exist today (`tests/src/Unit/**`).
- Unit tests may **not** touch the database, entity API, or `\Drupal::service()` — mock instead.
- When a change is testable in isolation, add or update a unit test in the same PR.
- Kernel / functional / browser tests are not currently scaffolded — do not invent them without asking.
- See [08-testing-guide.md](08-testing-guide.md) for the exact `phpunit` invocation and existing test file list.

## Diff hygiene

- Prefer `multi_replace_string_in_file` for related edits across files.
- Keep patches under ~200 lines unless the user asked for a large change.
- Never mix a functional fix with an unrelated stylistic edit in the same commit.
- One logical change per commit; commit message references the story/task if the project convention applies.

## Logging & error handling

- Errors on the CAPI wire are the norm (throttling, 5xx). Handle them explicitly — do not let raw exceptions surface to the queue worker.
- On unrecoverable error, use the existing pattern: set `haserror = 1`, `errormessage = <short>`, transition `status`/`file_upload_status` to a terminal-failure value, log with severity, and let the operator UI (`Reports → TMGMT_CONTENTAPI`) surface it.
- On transient error, back off (`HandleThrottling`) and let the queue retry.

## Configuration

- Translator settings schema is in [config/schema/tmgmt_contentapi.translator.schema.yml](../../../web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/config/schema/tmgmt_contentapi.translator.schema.yml).
- Any new setting must have a schema entry, a form control (typically on `AutoBundleForm` or `ContentApiTranslatorUI`), and a sensible default.

## Security

- CSRF: `QueueProcessController` route uses `_permission: 'access queue process'` and constants from `@csrf_token`. Follow that pattern for any new POST endpoint.
- Do not log secrets (CAPI token, JWT, provider credentials).
- Do not echo unfiltered response bodies to UI; they can contain user content.
