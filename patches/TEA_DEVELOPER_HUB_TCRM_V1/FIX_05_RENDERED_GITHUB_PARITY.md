# FIX 05 — Rendered GitHub Module Parity

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com/developer-hub`).

## Confirmed production failure

The current production page is connected to GitHub, but the rendered UI still shows only the connection/repository area plus the old generic Developer Hub cards. The following TCRM GitHub sections are NOT visible in the live page:

- Synchronization Status
- Push Mode control inside the GitHub workflow
- Review Push
- Review Pull
- Review Sync
- Safe Cleanup
- Review & Execute
- preview details / affected files
- commit message
- Execute Reviewed Action / Review and Push
- Git operation progress
- Git operation log

Previous reports claiming these were visible are not accepted unless they are verified in the actual production browser DOM.

## Required reference

Use the current TCRM Developer Hub reference already pinned by this bundle:

- repo: `mohamedamouseo-a11y/TCRM-MAIN-Tamiyouz-CRM-`
- commit: `78273711727e834ca88029e39ff0f6ae302d427a`
- UI reference: `client/src/components/DeveloperHubTab.tsx`

The TCRM reference visibly renders:

1. `Connection & Repository`
2. `Synchronization Status`
3. Push Mode (`off`, `review`, `auto`)
4. `Review Push`, `Review Pull`, `Review Sync`, `Safe Cleanup`
5. `Review & Execute`
6. Preview status/details, local/remote ahead, expected action, affected files, blockers/conflicts and Push Control checks
7. Commit message input
8. Explicit Execute button after a valid review
9. Progress percentage + current step
10. `Operation Log`

These controls stay rendered even when prerequisites are missing; unavailable controls are disabled rather than removed.

## Task

1. Inspect the live TEA route composition and determine exactly which component instance is rendered at `/developer-hub`.
2. Inspect `/opt/TBA/src/components/features/developer-hub/github-panel.tsx` and every parent component that conditionally mounts it.
3. Identify why production renders only the connection/branch portion despite the previous implementation report. Check for:
   - early return / conditional branch
   - duplicated or stale GitHub panel component
   - parent rendering a legacy/minimal panel
   - feature flag or connection-state conditional hiding the lower sections
   - component export/import mismatch
   - stale Vite module/runtime
4. Fix the actual rendered component, not an unused file.
5. Preserve the current working GitHub connection and backend APIs. Do not rewrite backend unless a frontend call required by the full UI is genuinely missing.
6. Render the full TCRM GitHub workflow in the production page. The GitHub workflow must be visually grouped as a complete module and must not be substituted by the generic Push Control card lower on the page.
7. Do not remove the existing generic Repository Status, Operations, Audit, MCP, AI Context or Source Export cards unless the TCRM parity implementation explicitly supersedes duplicate GitHub-only information.
8. Keep Super Admin authorization unchanged.
9. No new framework, service, dependency, port, nginx change or Git write.

## Mandatory production verification

Verification must inspect the ACTUAL rendered production browser page at:

`https://tea.tamiyouz.com/developer-hub`

Do not mark PASS by reading source code alone.

After loading the connected state, confirm the production DOM contains all of these visible labels at the same time:

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

Also confirm:

- repo selector exists
- branch selector exists
- disconnect exists
- review buttons are enabled/disabled only according to prerequisites
- Review & Execute remains visible before a review (empty-state text is acceptable)
- progress/log area remains visible in its TCRM-style workflow location
- page survives refresh and TEA service restart
- normal OpenHands UI still works

If any mandatory label/section is absent from the actual production DOM, STATUS must be FAILED.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_05_RENDERED_GITHUB_PARITY
LIVE_COMPONENT_PATH=
ROOT_CAUSE=
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
PRODUCTION_DOM_VERIFIED=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
RESTART_SURVIVAL=PASS|FAIL
CHANGED_FILES=<paths>
STATUS=READY|FAILED
```