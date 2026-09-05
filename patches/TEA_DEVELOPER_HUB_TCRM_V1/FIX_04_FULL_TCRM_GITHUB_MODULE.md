# FIX 04 — Full TCRM GitHub Module Parity

Target: NEW TEA only at `/opt/TBA` on `158.220.119.80`.

## Confirmed problem

The current TEA Developer Hub renders only the GitHub token/Connect block. The rest of the GitHub module is missing from the page.

This is NOT TCRM parity.

## Exact reference

Use the current TCRM Developer Hub implementation as the UI/behavior reference:

- repo: `mohamedamouseo-a11y/TCRM-MAIN-Tamiyouz-CRM-`
- pinned commit: `78273711727e834ca88029e39ff0f6ae302d427a`
- file: `client/src/components/DeveloperHubTab.tsx`

Do not invent a simplified replacement.

## Required GitHub module structure

Port the complete TCRM GitHub area into TEA using TEA/OpenHands UI primitives and the already-created TEA backend endpoints.

### A. Connection & Repository card

The card must visibly contain, even before GitHub is connected:

- GitHub Personal Access Token input
- Show / Hide token
- Verify / Connect action
- Repository selector
- Repository Refresh action
- Branch selector
- Branch Refresh action
- Save Selection action
- Disconnect action
- Saved Repository status block
- Saved Branch status block

IMPORTANT: Repository/branch controls may be disabled until prerequisites exist, but they must NOT be conditionally omitted from the UI. TCRM renders the structure and disables unavailable controls.

### B. Synchronization Status card

Render next to Connection & Repository, matching TCRM information architecture:

- Push Mode selector: Off / Review / Auto
- explanatory push-mode text
- sync status
- Local only commits
- GitHub only commits
- Review Push
- Review Pull
- Review Sync
- Safe Cleanup
- last sync metadata when available

This card must remain visible before connection/review, with disabled actions where required. Do not hide it.

### C. Review & Execute card

Always render the Review & Execute section below the two GitHub cards.

Before a preview exists, show the empty/default state (choose a review action above).

After preview, show the TCRM-equivalent review detail including:

- selected repo + branch
- sync-state badge
- expected action
- local branch
- affected file count
- Push Control results: Verify / Tests / Build / Security / Remote Sync
- blockers/conflicts
- affected files list with direction/status
- cleanup details when action is cleanup
- commit message input
- Execute Reviewed Action / Review and Push button
- stop-monitoring action while operation is running
- retry review on recoverable/blocked failure

### D. Progress + Operation Log

Render the TCRM-equivalent operation progress section and terminal-style Operation Log:

- current step
- percentage
- progress bar
- sanitized log lines
- no secret/token values
- recovery-safe behavior already present in backend must remain wired to UI

## Behavior rules

- Full module must be rendered for Super Admin even when GitHub is disconnected.
- Disconnected state disables repo/branch/sync actions but does not remove them.
- Connecting GitHub must populate repositories without page reload.
- Selecting a repository loads branches without page reload.
- Save Selection updates the saved repo/branch blocks.
- Review actions must use the existing preview endpoint and fingerprint integrity flow.
- Execute remains disabled until a valid non-stale review passes.
- Existing Super Admin server-side guard remains unchanged.
- Do not weaken stale-review protection.
- Do not expose token after submission.
- Do not introduce another backend/service/port.
- Do not change dependencies, nginx, systemd, or unrelated OpenHands UI.
- Do not perform Git writes while applying this patch.

## Scope clarification

This patch is specifically about restoring the complete GitHub module UI/interaction. Do not spend time redesigning unrelated Developer Hub cards.

## Verification

Verify at `https://tea.tamiyouz.com/developer-hub` in both states where practical:

1. disconnected GitHub state
2. connected GitHub state

The disconnected page itself must visibly show:

- token/connect
- repository selector
- branch selector
- save/disconnect controls
- synchronization status card
- Push Mode
- Review Push/Pull/Sync/Safe Cleanup controls
- Review & Execute card
- Operation Log

Controls can be disabled; sections cannot be missing.

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_04_FULL_GITHUB_MODULE
CONNECTION_REPOSITORY_FULL_UI=PASS|FAIL
REPO_SELECTOR_VISIBLE_WHEN_DISCONNECTED=PASS|FAIL
BRANCH_SELECTOR_VISIBLE_WHEN_DISCONNECTED=PASS|FAIL
SYNC_STATUS_CARD=PASS|FAIL
PUSH_MODE_UI=PASS|FAIL
REVIEW_ACTIONS_VISIBLE=PASS|FAIL
SAFE_CLEANUP_VISIBLE=PASS|FAIL
REVIEW_EXECUTE_CARD=PASS|FAIL
PREVIEW_DETAILS=PASS|FAIL
PROGRESS_UI=PASS|FAIL
OPERATION_LOG=PASS|FAIL
CONNECT_POPULATES_REPOS=PASS|FAIL
REPO_LOADS_BRANCHES=PASS|FAIL
STALE_REVIEW_PROTECTION=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
DOMAIN_DEVHUB_UI=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=<paths>
STATUS=READY|FAILED
```