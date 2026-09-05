# FIX 10 — GitHub Panel TSX Parse Error Recovery

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com/developer-hub`).

## Confirmed production failure

The public browser currently shows Vite transform failure:

- file: `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`
- location: around line 16
- error: `Unexpected token`
- failing statement starts with: `const { t } = useTranslation("openhands");`

This means FIX 09 cannot be accepted as READY: the live TSX module does not parse, so no browser-visible parity claim is valid.

## Task

1. Inspect the first ~80 lines of the live `github-panel.tsx` and identify the exact malformed TSX/JavaScript structure around the component declaration, imports, temporary marker, braces, or duplicated statements.
2. Fix ONLY the syntax/structure needed to make the existing full panel compile. Do not rewrite the module again unless strictly necessary.
3. Remove any temporary FIX09 source-identity marker/code that broke or polluted the component after source identity was proven.
4. Preserve the full TCRM GitHub workflow already implemented: Connection & Repository, Synchronization Status, Push Mode, Review Push/Pull/Sync, Safe Cleanup, Review & Execute, Commit message, Progress, Operation Log.
5. Run the repository's existing frontend syntax/type/build validation relevant to this file BEFORE restarting anything. A Vite browser check alone is not enough.
6. Restart only the NEW TEA service if required after validation.
7. Verify the actual public browser page after restart. The Vite error overlay must be gone.
8. Do not touch nginx, ports, backend auth, GitHub credentials, other services, or Git.

## Mandatory verification

Do not mark PASS unless all are true:

- `github-panel.tsx` parses successfully
- relevant typecheck/build check passes for the touched code (or only unrelated pre-existing failures remain and are explicitly identified)
- no Vite transform overlay in production
- `/developer-hub` renders
- all full GitHub workflow sections remain visible
- normal OpenHands home UI still works
- service restart survives

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_10_GITHUB_PANEL_PARSE_ERROR
ROOT_CAUSE=
PARSE_ERROR_FIXED=PASS|FAIL
TEMP_MARKER_REMOVED=YES|NO
TYPECHECK_BUILD=<result>
VITE_OVERLAY_GONE=PASS|FAIL
CONNECTION_REPOSITORY_VISIBLE=PASS|FAIL
SYNC_STATUS_VISIBLE=PASS|FAIL
PUSH_MODE_VISIBLE=PASS|FAIL
REVIEW_ACTIONS_VISIBLE=PASS|FAIL
REVIEW_EXECUTE_VISIBLE=PASS|FAIL
COMMIT_MESSAGE_VISIBLE=PASS|FAIL
OPERATION_LOG_VISIBLE=PASS|FAIL
DOMAIN_DEVHUB_UI=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
RESTART_SURVIVAL=PASS|FAIL
CHANGED_FILES=<paths>
STATUS=READY|FAILED
```
