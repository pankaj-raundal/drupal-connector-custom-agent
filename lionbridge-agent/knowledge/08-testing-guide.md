# 08 — Testing guide

## What exists today

Only **unit tests**, under `tmgmt_contentapi/tests/src/Unit/`:

- `EventSubscriber/WorkflowGateSubscriberTest.php`
- `Services/AnalysisCodeApiTest.php`
- `Services/AutoBundleGroupKeyBuilderTest.php`
- `Form/AutoBundleFormTest.php`
- `Services/AnalysisCodeApiTest.md` (companion notes)

There is **no** kernel-test or functional-test scaffolding in the module today. Do not invent one silently.

## Running unit tests

From the workspace root:

```bash
SIMPLETEST_BASE_URL=http://localhost \
SIMPLETEST_DB=sqlite://localhost//dev/null \
php vendor/bin/phpunit -c web/core/phpunit.xml.dist \
  web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/tests/src/Unit
```

Single file:

```bash
php vendor/bin/phpunit -c web/core/phpunit.xml.dist \
  web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/tests/src/Unit/Form/AutoBundleFormTest.php
```

## Rules

- Unit tests **must not** hit the DB, `\Drupal::service()`, entity API, or file system. Mock.
- Extend `Drupal\Tests\UnitTestCase` (already the pattern in existing tests).
- Use PHPUnit mock objects; no `prophecy` unless already present in the file.
- One assertion cluster per behavior. Name test methods `testWhenXThenY`.

## What to test per category

| Category | Prefer testing |
|---|---|
| SEND / IMPORT / STATE | Extract pure logic (state transition decisions, filename composition, group-key builder logic) into a testable method and test that. Do NOT try to unit-test DB queries — cover those via manual repro + logs. |
| AUTOBUNDLE | Group-key determinism, trigger evaluator threshold arithmetic, priority-tier resolution with the `priority_empty_fallback` variants. |
| SCHEMA | No test — schema hooks are validated by `drush updatedb` locally. Add a manual verification note in the PR. |
| UI | Form-array shape and `#states` conditionals via a mocked form state (see `AutoBundleFormTest.php` as the pattern). |
| COMPAT | Run the whole unit suite on the target PHP version. |

## Regression tests to add when you fix something

If you fix an issue in [04-known-issues-safeguards.md](04-known-issues-safeguards.md), add a unit test that would have failed *before* the fix. If the fix is DB-shaped and not unit-testable, document the manual repro in the PR body.

## Manual repro procedure (template)

When unit-testing isn't feasible:

```
### Repro
1. …
2. …

### Expected
- …

### Observed (pre-fix)
- …

### Verification (post-fix)
- drush ev "…" prints …
- SELECT status, statuscode, file_upload_status, source_site FROM tmgmt_capi_request_processor WHERE tjid = <id> shows …
```
