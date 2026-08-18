---
mode: 'lionbridge-maintainer'
description: 'Focused code review of a Lionbridge Content API diff / branch / PR.'
---

# Review a Lionbridge Content API change

## Inputs I will provide

- A diff, a branch name, a PR link, or a list of changed files.
- The story/task id if the project convention applies (`@dai-story: SNN`).

## Procedure

1. **Enumerate** the changed files. Group by category (SEND / IMPORT / STATE / AUTOBUNDLE / SCHEMA / UI / COMPAT).
2. For each category, load only the matching knowledge files from [.github/lionbridge-agent/README.md](../lionbridge-agent/README.md).
3. **Read the diff** carefully. Do NOT read the full files — read the diff hunks plus a small window of context (`grep_search` for the enclosing function name if needed).
4. Run the [10-review-checklist.md](../lionbridge-agent/knowledge/10-review-checklist.md) end to end.
5. For each unchecked box, produce a specific review comment with:
   - File + line range (as a workspace-relative link).
   - Exact concern (map to a safeguard in [04-known-issues-safeguards.md](../lionbridge-agent/knowledge/04-known-issues-safeguards.md) or an invariant in [07-do-not-break.md](../lionbridge-agent/knowledge/07-do-not-break.md)).
   - Concrete suggested change.
6. Group findings as **Blocking**, **Should-fix**, **Nit**.
7. Also emit:
   - **Flow coverage matrix** (ZIP / non-ZIP / single / bulk / multi-container / continuous / one-off) — mark each row impacted / unchanged.
   - **Regression risk vs historical issues** (S1–S10 from [04](../lionbridge-agent/knowledge/04-known-issues-safeguards.md)).
   - **Test coverage** — is a matching unit test present?
   - **Compat verdict** — PHP 8.1–8.4 and Drupal 9/10/11.

## Do not

- Do not rewrite the change. Suggest, do not commit.
- Do not open files outside the diff unless a specific finding requires it.
- Do not mark a finding blocking without naming the exact safeguard or invariant it violates.
