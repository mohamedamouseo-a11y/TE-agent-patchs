# FIX 09 — Production Source Identity + Real Full GitHub Module

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com/developer-hub`).

## Confirmed reality from user screenshot

The production browser still renders the legacy/minimal Developer Hub layout. At the bottom of the actual page, the generic cards appear immediately after the small GitHub/Branch card. The required full TCRM GitHub workflow is NOT present.

Therefore previous PASS reports are invalid. Do not trust source grep, TypeScript checks, route imports, testids, curl HTML, or an internal browser assertion unless the exact source identity served to the public browser is proven.

## Goal

Find which exact source module the public browser is executing for `/developer-hub`, then make that exact module render the full TCRM GitHub workflow.

## Mandatory diagnostic before editing

1. Inspect nginx upstream for `tea.tamiyouz.com`.
2. Inspect the exact ingress process/PID/CWD/command.
3. Inspect the exact Vite process/PID/CWD/command/port.
4. From the PUBLIC page HTML/module graph, trace the loaded route module for `/developer-hub` and the loaded GitHub panel module.
5. Resolve all aliases (`#`, `@`, tsconfig/vite aliases) to absolute filesystem paths.
6. Search all source trees under `/opt/TBA` for:
   - `DeveloperHubGitHubPanel`
   - `GitHub Connection`
   - `Push control`
   - `Source export`
   - `Synchronization Status`
   - `Review & Execute`
7. Report every duplicate candidate file and identify which one is actually requested/executed by the browser.

## Hard identity proof

Before implementing the final fix, add a TEMPORARY visible marker to the suspected live full-panel root:

`TEA DEVHUB LIVE MODULE FIX09`

Reload the PUBLIC domain in a real browser.

- If the marker does NOT visibly appear, you have the wrong file. Revert that temporary marker and continue tracing.
- Do not proceed until the marker appears on the public browser page.

Once the correct file is proven, remove the temporary marker and implement the final UI there.

## Final UI requirement

The exact live module must visibly render, in the connected state, all of these sections BEFORE the generic cards:

1. Connection & Repository
2. Synchronization Status
3. Push Mode (`off`, `review`, `auto`)
4. Review Push
5. Review Pull
6. Review Sync
7. Safe Cleanup
8. Review & Execute
9. Commit message
10. Operation Log

The generic Repository Status / Operations / Audit / Push Control / MCP / AI Context / Source Export cards may remain below the full GitHub module.

Do not hide the full GitHub workflow based on connection state. Missing prerequisites should disable controls, not remove sections.

## Implementation constraints

- Preserve existing working GitHub connection and stored token.
- Preserve Super Admin authorization.
- No new service/framework/dependency/port/nginx change unless the diagnosis proves the public domain points to the wrong already-existing TEA upstream.
- No Git writes.
- Do not touch the old TEAgent server.
- Change the minimum number of files.

## Mandatory browser verification

After the fix, use a real browser on:

`https://tea.tamiyouz.com/developer-hub`

Capture TWO screenshots:

A. Upper section showing:
- Connection & Repository
- Synchronization Status
- Push Mode
- Review Push / Pull / Sync / Safe Cleanup

B. Lower section showing:
- Review & Execute
- Commit message
- Operation Log
- beginning of generic cards below it

A PASS is forbidden unless those exact visible labels are present in the captured screenshots.

Also verify normal OpenHands home UI still works.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_09_PRODUCTION_SOURCE_IDENTITY
NGINX_UPSTREAM=
INGRESS_PID=
INGRESS_CWD=
VITE_PID=
VITE_CWD=
PUBLIC_ROUTE_MODULE_URL=
PUBLIC_PANEL_MODULE_URL=
LIVE_ROUTE_ABSOLUTE_PATH=
LIVE_PANEL_ABSOLUTE_PATH=
DUPLICATE_ROUTE_FILES=<paths|NONE>
DUPLICATE_PANEL_FILES=<paths|NONE>
TEMP_MARKER_PUBLICLY_VISIBLE=PASS|FAIL
ROOT_CAUSE=
FIX_APPLIED=
CONNECTION_REPOSITORY_VISIBLE=PASS|FAIL
SYNC_STATUS_VISIBLE=PASS|FAIL
PUSH_MODE_VISIBLE=PASS|FAIL
REVIEW_ACTIONS_VISIBLE=PASS|FAIL
REVIEW_EXECUTE_VISIBLE=PASS|FAIL
COMMIT_MESSAGE_VISIBLE=PASS|FAIL
OPERATION_LOG_VISIBLE=PASS|FAIL
SCREENSHOT_UPPER_PATH=
SCREENSHOT_LOWER_PATH=
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=<paths>
STATUS=READY|FAILED
```

If the temporary marker cannot be made visible on the public domain, STATUS=FAILED and do not claim the full module is live.