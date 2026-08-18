# Lionbridge Translation Provider — Custom Copilot Agent

A specialized VS Code / GitHub Copilot chat-mode agent that maintains, debugs, reviews, and releases the Drupal module [`lionbridge_translation_provider`](https://www.drupal.org/project/lionbridge_translation_provider) — specifically its `tmgmt_contentapi` submodule at [`web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/`](../web/sites/default/modules/contrib/lionbridge_translation_provider/tmgmt_contentapi/).

Module version at time of writing: **9.4.7**.

## Directory layout

```
.github/
├── chatmodes/
│   └── lionbridge-maintainer.chatmode.md      ← the agent (loaded when the mode is picked)
├── prompts/
│   ├── lb-debug.prompt.md                     ← /lb-debug
│   ├── lb-fix.prompt.md                       ← /lb-fix
│   ├── lb-review.prompt.md                    ← /lb-review
│   ├── lb-release-prep.prompt.md              ← /lb-release-prep
│   └── lb-test.prompt.md                      ← /lb-test
├── lionbridge-agent/
│   ├── README.md                              ← knowledge router (category → files to load)
│   └── knowledge/                             ← progressive-load knowledge base (do NOT auto-load)
│       ├── 01-architecture.md
│       ├── 02-workflow-state-machine.md
│       ├── 03-database-schema.md
│       ├── 04-known-issues-safeguards.md
│       ├── 05-release-history.md
│       ├── 06-development-rules.md
│       ├── 07-do-not-break.md
│       ├── 08-testing-guide.md
│       ├── 09-file-map.md
│       └── 10-review-checklist.md
└── copilot-instructions.md                    ← (pre-existing) global instruction file
```

## How to invoke

### 1. Development (writing a fix)

1. In the VS Code Copilot Chat picker, choose the **`lionbridge-maintainer`** chat mode.
2. Optionally start with `/lb-debug` if the symptom isn't diagnosed yet.
3. When the root cause is known, run `/lb-fix` with the target file/line range.
4. When a change is testable in isolation, run `/lb-test`.

### 2. Debugging (unknown symptom)

1. Choose **`lionbridge-maintainer`** chat mode.
2. Run `/lb-debug`.
3. Provide symptom, log snippet, `tjid`, and reproducibility.
4. Follow up with `/lb-fix` if a code change is authorized, or execute the SQL/drush recovery procedure the agent produces.

### 3. Code review (diff / PR)

1. Choose **`lionbridge-maintainer`** chat mode.
2. Run `/lb-review`.
3. Point the agent at the diff or list of changed files.
4. The agent emits blocking / should-fix / nit findings, a flow-coverage matrix, and a compat verdict.

### 4. Release preparation

1. Choose **`lionbridge-maintainer`** chat mode.
2. Run `/lb-release-prep` with the target version.
3. The agent bumps `tmgmt_contentapi.info.yml`, sweeps regressions S1..S10, runs unit tests, and emits release notes.

## Token efficiency — how this design stays small

Traditional approach: one huge system prompt containing everything about the repo → every request pays the full token cost, and the agent still misses relevant nuance because the prompt overflows the working attention budget.

**This design uses "progressive context loading":**

1. **Chatmode file is small.** [lionbridge-maintainer.chatmode.md](chatmodes/lionbridge-maintainer.chatmode.md) contains only the identity, the golden rules, the classifier, the workflow, the DB/concurrency reminders, the output template, and a fast tool cheatsheet. It intentionally does NOT contain the schema, the state machine, the release history, or the file map.
2. **Knowledge is split into 10 small files** ([lionbridge-agent/knowledge/](lionbridge-agent/knowledge/)), each single-topic. They are **not** auto-applied via an `applyTo:` glob — they are loaded by name only when the classifier decides the topic is relevant.
3. **A routing table** in [lionbridge-agent/README.md](lionbridge-agent/README.md) maps each of 10 categories (SEND / IMPORT / STATE / AUTOBUNDLE / CONTINUOUS / SCHEMA / UI / COMPAT / REVIEW / RELEASE) to the minimum knowledge files and source files needed. The agent looks up the row for the classified category and loads only those.
4. **Prompts are thin wrappers** over the chatmode. Each prompt names the procedure and pins the *first* files to load — no duplication of the knowledge base itself.
5. **Source files are read with `grep_search` first, then `read_file` on a narrow line range.** The chatmode explicitly forbids reading files >500 lines end-to-end.
6. **The `do-not-break` file is deliberately tiny** so it can be loaded on every task without cost.
7. **The output template forces the model to name what it loaded and why**, which discourages redundant re-reads within a task.

Net effect: the average request loads the chatmode (~5 kB) + the do-not-break list (~2 kB) + one or two topic files (~3–6 kB) + narrow slices of at most 2–4 source files — well under 20 kB of context vs 200+ kB for the whole module.

## Design principles enforced by the agent

- **Understand before modifying.** No edit without a prior read.
- **Smallest safe change.** No refactor drift.
- **Root cause first.** Bandaid fixes are explicitly rejected.
- **Protect existing safeguards.** Every historical safeguard (S1..S10 in [04-known-issues-safeguards.md](lionbridge-agent/knowledge/04-known-issues-safeguards.md)) is listed as an invariant in [07-do-not-break.md](lionbridge-agent/knowledge/07-do-not-break.md).
- **Drupal 9/10/11 + PHP 8.1–8.4 compatibility.** Enforced in the review checklist.
- **DB/concurrency vigilance.** Mandatory section in the chatmode.
- **Explicit flow coverage.** ZIP / non-ZIP / single / bulk / multi-container / continuous / one-off are named in the output template.
- **Regression check.** The review and release prompts sweep the historical issues list.
- **Targeted searches.** Cheatsheet + rules discourage repository-wide exploration.
- **Minimal patches.** ~200-line soft cap unless explicitly waived.
- **Tests recommended or added.** Unit-test guide is included; kernel/functional test scaffolding will not be silently invented.
- **Focused code review.** The [10-review-checklist.md](lionbridge-agent/knowledge/10-review-checklist.md) is the mandated final step.
- **Structured explanation.** Section-4 output template is enforced.

## Honesty about unknowns

The agent is instructed to mark any claim it cannot confirm from code as `UNKNOWN — verify`. Items in this delivery that are already flagged unknown:

- **Exact minor version where several safeguards landed** (S1 multi-container, S3 source_site, S8 memory/timeout). [05-release-history.md](lionbridge-agent/knowledge/05-release-history.md) lists these as UNKNOWN.
- **`MEMORY_TIMEOUT_FIX.md` is 0 bytes** in the workspace, so its exact content is not reflected in [04-known-issues-safeguards.md](lionbridge-agent/knowledge/04-known-issues-safeguards.md); it is called out as needing verification.
- **`file_upload_status` value set** — the install file names `PENDING`, `UPLOADING`, `UPLOADED`, `FAILED`, but grep found `FILE_GENERATING` and `READY_FOR_UPLOAD` in queries at ~L1373 of `CapiDataProcessor`. [02-workflow-state-machine.md](lionbridge-agent/knowledge/02-workflow-state-machine.md) records both sets and marks the extended set as observed but not authoritatively documented.
- **Exact status→status transitions** in [02-workflow-state-machine.md](lionbridge-agent/knowledge/02-workflow-state-machine.md) are labelled `INFERRED FROM CODE` where they were reconstructed from filter clauses rather than a single documented state machine.
- **Precise line-ranges** cited in knowledge files (e.g. "L472–L595 in `CapiDataProcessor`") were correct at 9.4.7; they may drift with future edits and should be reconfirmed by `grep_search` before being trusted for a fix.
- **PHP version pin for TMGMT** — TMGMT dependency has no version pin visible in the info.yml; assumed compatible with the same core range.

## Extending the agent

- **New safeguard landed?** Append a row to [04-known-issues-safeguards.md](lionbridge-agent/knowledge/04-known-issues-safeguards.md) *and* to [07-do-not-break.md](lionbridge-agent/knowledge/07-do-not-break.md).
- **New schema element?** Update [03-database-schema.md](lionbridge-agent/knowledge/03-database-schema.md) and add a row to [05-release-history.md](lionbridge-agent/knowledge/05-release-history.md).
- **New service/queue-worker/subscriber?** Update [01-architecture.md](lionbridge-agent/knowledge/01-architecture.md) and [09-file-map.md](lionbridge-agent/knowledge/09-file-map.md).
- **New category of task?** Add a row to the routing table in [lionbridge-agent/README.md](lionbridge-agent/README.md); do NOT expand the chatmode's classifier without also listing the files to load.
- **Never** dump raw source code into a knowledge file; describe it and link to it.
