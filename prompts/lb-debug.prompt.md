---
mode: 'lionbridge-maintainer'
description: 'Diagnose a Lionbridge Content API bug (SENDING stuck, IN_QUEUE stuck, orphan rows, redelivery, cross-env, upload failure, etc.).'
---

# Debug a Lionbridge Content API issue

Follow the standard workflow in the [lionbridge-maintainer chatmode](../chatmodes/lionbridge-maintainer.chatmode.md).

## Inputs I will provide

- **Symptom** (what the operator sees).
- **Where observed** (log line, DB row snapshot, TMGMT job id, CAPI request id).
- **When it started** (release, migration, DB restore, container change).
- **Reproducibility** (always / intermittent / after specific action).

If any of these are missing, ask **one** targeted question before diagnosing.

## Debug procedure

1. **Classify** the symptom (SEND / IMPORT / STATE / AUTOBUNDLE / CONTINUOUS).
2. **Load** only the knowledge files listed for that category in [.github/lionbridge-agent/README.md](../lionbridge-agent/README.md).
3. **Trace** the row: given a `tjid`, ask me to run:
   ```sql
   SELECT rid, tjid, tjiid, updateid, jobid, requestid,
          status, statuscode, file_upload_status, file_upload_attempts,
          haserror, errormessage, source_site, updatedtime, lastupdated
     FROM tmgmt_capi_request_processor WHERE tjid = <id>
     ORDER BY rid;
   ```
4. **Cross-reference** against [02-workflow-state-machine.md](../lionbridge-agent/knowledge/02-workflow-state-machine.md):
   - Does the row combination match a *known* stuck-state pattern from [04-known-issues-safeguards.md](../lionbridge-agent/knowledge/04-known-issues-safeguards.md)?
   - Which safeguard *should* have prevented it, and why didn't it fire?
5. **Locate** the exact code path via targeted `grep_search` for the observed status literals (e.g. `'SENDING'`, `'IN_QUEUE'`, `'PENDING'`) inside the module.
6. **State root cause** in ≤5 sentences.
7. **Propose a minimal fix** OR a recovery procedure (SQL + `drush` commands) if the fix requires a code change I have not been asked to make yet.
8. Emit the section-4 output template.

## Do not

- Do not run `drush` commands that mutate DB rows without explicit approval.
- Do not propose "add a try/catch and log" as a fix — that is a bandaid, not a root cause.
- Do not read files >500 lines end-to-end; use `grep_search` to jump to line ranges.
