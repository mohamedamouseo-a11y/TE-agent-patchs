# FIX 06 — Live Runtime Binding Truth

Target only the NEW TEA instance:

- source: `/opt/TBA`
- domain: `https://tea.tamiyouz.com/developer-hub`

## Confirmed failure from actual browser screenshot

The live browser is still rendering the old/minimal Developer Hub layout.

Even after FIX 05 reported PASS, the bottom of the real production page contains only:

- Repository Status
- minimal GitHub connection/branch card
- Operations
- Audit
- generic Push control
- MCP control
- AI context
- Source export

The actual browser page still does **NOT** show:

- Synchronization Status
- Push Mode selector inside GitHub workflow
- Review Push
- Review Pull
- Review Sync
- Safe Cleanup
- Review & Execute
- Commit message
- Operation Log

Therefore the previous `PRODUCTION_DOM_VERIFIED=PASS` report is invalid.

## Goal

Do not redesign anything. Find why the browser is not executing/rendering the full `github-panel.tsx` implementation and make the LIVE production browser load that exact component.

## Mandatory diagnosis before changing code

1. Identify the exact Vite process serving `tea.tamiyouz.com`:
   - PID
   - CWD
   - command
   - frontend port
   - start time

2. Confirm nginx/ingress path from domain to that Vite process.

3. On disk, inspect:
   - `/opt/TBA/src/routes/developer-hub.tsx`
   - `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`

4. Confirm the on-disk `github-panel.tsx` contains all mandatory labels:
   - `Connection & Repository`
   - `Synchronization Status`
   - `Push Mode`
   - `Review Push`
   - `Review Pull`
   - `Review Sync`
   - `Safe Cleanup`
   - `Review & Execute`
   - `Commit message`
   - `Operation Log`

5. Fetch the exact frontend module/assets being served through the live domain and determine whether the LIVE browser bundle/module contains the same mandatory labels.

6. Determine whether the problem is one of:
   - stale Vite runtime
   - stale optimized dependency/module cache
   - wrong source tree/CWD
   - wrong route import
   - duplicate legacy github panel
   - old process still bound to frontend port
   - parent component rendering the legacy/minimal panel
   - failed compile causing Vite to preserve previous module

## Fix rules

- Apply only the smallest confirmed fix.
- If runtime is stale, restart only the NEW TEA frontend/service.
- If Vite cache is stale, clear only TEA Vite cache/artifacts required to reload source.
- If wrong import/component is active, fix only that binding.
- If duplicate legacy component is active, point the route to the full component and remove only the obsolete binding if safe.
- Do not modify backend APIs unless the live frontend proves they are required.
- No dependency upgrade.
- No new service/port.
- No Git writes.
- Do not touch the old TEAgent server.

## Verification must use a REAL browser

Use a real browser automation session against:

`https://tea.tamiyouz.com/developer-hub`

The connected GitHub state must be loaded.

Scroll the page and verify all of these labels are visually present in the rendered page at the same time:

- `Connection & Repository`
- `Synchronization Status`
- `Push Mode`
- `Review Push`
- `Review Pull`
- `Review Sync`
- `Safe Cleanup`
- `Review & Execute`
- `Commit message`
- `Operation Log`

Do not mark PASS from source grep, curl JSON, API status, or static code inspection alone.

If available, capture one browser screenshot showing `Synchronization Status` and another showing `Review & Execute` + `Operation Log`.

Also verify normal OpenHands home/conversation UI still works after the fix.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_06_LIVE_RUNTIME_BINDING
LIVE_VITE_PID=
LIVE_VITE_CWD=
LIVE_VITE_COMMAND=
LIVE_FRONTEND_PORT=
DISK_FULL_PANEL_LABELS=PASS|FAIL
LIVE_SERVED_FULL_PANEL_LABELS=PASS|FAIL
WRONG_SOURCE_TREE=YES|NO
STALE_RUNTIME=YES|NO
DUPLICATE_LEGACY_PANEL=YES|NO
ROUTE_BINDING_CORRECT=PASS|FAIL
ROOT_CAUSE=
FIX_APPLIED=
SYNC_STATUS_BROWSER_VISIBLE=PASS|FAIL
REVIEW_ACTIONS_BROWSER_VISIBLE=PASS|FAIL
REVIEW_EXECUTE_BROWSER_VISIBLE=PASS|FAIL
OPERATION_LOG_BROWSER_VISIBLE=PASS|FAIL
REAL_BROWSER_VERIFIED=PASS|FAIL
SCREENSHOTS_CAPTURED=YES|NO
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=<paths>
STATUS=READY|FAILED
```

If any browser-visible section is missing, `STATUS=FAILED`.