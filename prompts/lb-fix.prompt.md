---
mode: 'lionbridge-maintainer'
description: 'Implement a minimal, safe fix in tmgmt_contentapi and add a matching test.'
---

# Fix a Lionbridge Content API issue

## Inputs I will provide

- Diagnosed **root cause** (from `/lb-debug` or from me).
- **Target file(s)** and, if I have them, **line ranges**.
- **Affected flows** I care about (defaults: all, per the section-4 template).

## Procedure

1. Re-verify the root cause by reading only the target lines. Do not expand the search unless evidence forces you to.
2. Consult [07-do-not-break.md](../lionbridge-agent/knowledge/07-do-not-break.md). Confirm the fix does not touch any protected invariant.
3. Draft the smallest patch. Prefer editing existing code paths over adding new services/subscribers/hooks.
4. If schema changes are needed:
   - Add a new `tmgmt_contentapi_update_9XXX()` hook with idempotent guards.
   - Update the corresponding schema helper function so fresh installs match.
   - Never edit hooks numbered `9100`–`9110` (or any historical hook that has run in prod).
5. Add or update a **unit test** if the change is testable in isolation (see [08-testing-guide.md](../lionbridge-agent/knowledge/08-testing-guide.md)). If it is not unit-testable, add a manual repro block instead.
6. Run the review checklist ([10-review-checklist.md](../lionbridge-agent/knowledge/10-review-checklist.md)) mentally and mark each item.
7. Emit the section-4 output template.

## Constraints

- Total diff should stay under ~200 lines unless I explicitly authorized larger.
- No refactors, no renames, no docblock-only edits.
- Do not touch `web/modules/contrib/tmgmt/**` or `web/modules/contrib/lionbridge_translation_provider/**`.
- Do not hand-edit `src/Swagger/Client/**`.
- Every new status literal → constant on `CapiDataProcessor`.
- Every cross-container memory → DB, not `\Drupal::state()`.
