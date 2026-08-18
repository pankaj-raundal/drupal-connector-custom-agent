---
mode: 'lionbridge-maintainer'
description: 'Add or update a unit test in tmgmt_contentapi/tests/src/Unit.'
---

# Add / update a Lionbridge unit test

## Inputs I will provide

- The behavior to cover (ideally the root cause you already diagnosed).
- The target class or method.

## Procedure

1. Load [08-testing-guide.md](../lionbridge-agent/knowledge/08-testing-guide.md).
2. Confirm the target is unit-testable: no DB, no `\Drupal::service()`, no entity API, no filesystem.
3. If it is not unit-testable, **stop** and emit a manual repro block instead.
4. If similar tests already exist (`tests/src/Unit/**`), match their style:
   - Extend `Drupal\Tests\UnitTestCase`.
   - Use PHPUnit mocks (`createMock`, `->method(...)->willReturn(...)`).
   - Name methods `testWhenXThenY`.
5. Add / update the test file. Keep it under ~200 lines.
6. Show me the command to run just that file (see [08-testing-guide.md](../lionbridge-agent/knowledge/08-testing-guide.md)).
7. If asked, run the test via terminal and report pass/fail.

## Do not

- Do not add kernel/functional/browser test scaffolding.
- Do not modify the class under test just to make it more testable, unless the extraction is small (a pure static/private method → protected, or a fold-out of a pure calculation into a new small method) AND the extraction is part of an already-approved refactor.
- Do not silently add new PHPUnit dependencies to `composer.json`.
