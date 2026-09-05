# FIX 08 — Force Full GitHub Module Mount

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com/developer-hub`).

## Production evidence

The latest real browser screenshot still reaches the bottom of `/developer-hub` and shows only the legacy/generic cards:

- Repository Status
- minimal GitHub connection/branch area
- Operations
- Audit
- Push control
- MCP control
- AI context
- Source export

The required full TCRM GitHub workflow is still NOT rendered in the actual page.

Previous source/DOM PASS reports are therefore not accepted.

## Required fix

1. Inspect `/opt/TBA/src/routes/developer-hub.tsx` and the actual route tree used by the running React Router app.
2. Identify the exact JSX that renders the current minimal GitHub card.
3. Import the full component explicitly from:
   `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`
4. Mount `<DeveloperHubGitHubPanel />` directly in the live `/developer-hub` route at top level, immediately after the page heading/status area and before the generic Operations/Audit/Push Control/MCP/AI/Source Export cards.
5. Remove only the legacy/minimal GitHub summary card if it duplicates the full module. Do NOT remove unrelated generic cards.
6. Do not hide the full panel based on connection state. It must always render; unavailable controls are disabled.
7. Ensure there is exactly ONE live GitHub panel implementation on this route.
8. Add `data-testid="developer-hub-github-full-module"` to the full panel root so the live browser can prove the correct component is mounted.
9. Do not touch backend, auth, ports, nginx, dependencies, Git config, or other services unless strictly required to reload the frontend.
10. Restart only the NEW TEA frontend/service if required.

## Mandatory live verification

Verify in a real browser at:
`https://tea.tamiyouz.com/developer-hub`

The production page must visibly contain ALL of these at the same time:

- Connection & Repository
- Synchronization Status
- Push Mode
- Review Push
- Review Pull
- Review Sync
- Safe Cleanup
- Review & Execute
- Commit message
- Operation Log

Also verify:

- `[data-testid="developer-hub-github-full-module"]` exists exactly once in the production DOM
- the old minimal GitHub-only card is not duplicated beside it
- the existing Repository Status / Operations / Audit / Push Control / MCP / AI Context / Source Export cards still render
- normal OpenHands home/conversation UI still works

You MUST capture a production browser screenshot after the fix. Do not report READY without a screenshot captured after the final reload.

If the route source appears correct but the live browser still lacks the module, continue tracing React Router route resolution/import resolution and Vite module serving until the actual mounted route is corrected. Do not stop at source grep.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_08_FORCE_FULL_GITHUB_ROUTE_MOUNT
LIVE_ROUTE_FILE=
LEGACY_GITHUB_CARD_FOUND=YES|NO
FULL_PANEL_IMPORT=
FULL_PANEL_MOUNT_COUNT=
TESTID_COUNT=
CONNECTION_REPOSITORY_VISIBLE=PASS|FAIL
SYNC_STATUS_VISIBLE=PASS|FAIL
PUSH_MODE_VISIBLE=PASS|FAIL
REVIEW_PUSH_VISIBLE=PASS|FAIL
REVIEW_PULL_VISIBLE=PASS|FAIL
REVIEW_SYNC_VISIBLE=PASS|FAIL
SAFE_CLEANUP_VISIBLE=PASS|FAIL
REVIEW_EXECUTE_VISIBLE=PASS|FAIL
COMMIT_MESSAGE_VISIBLE=PASS|FAIL
OPERATION_LOG_VISIBLE=PASS|FAIL
GENERIC_CARDS_PRESERVED=PASS|FAIL
SCREENSHOT_CAPTURED=YES|NO
SCREENSHOT_PATH=
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=<paths>
STATUS=READY|FAILED
```
