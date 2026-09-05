# TEA Developer Hub — GitHub UI TCRM Parity V1

Apply only to the NEW TEA instance at `/opt/TBA` on server `158.220.119.80`.

## Goal

Replace the current read-only GitHub status card in `/developer-hub` with the **full interactive GitHub module matching TCRM Main Developer Hub behavior and UX as closely as possible**, while keeping TEA's existing architecture, Super Admin security and manual-human Git policy.

## Exact TCRM reference

Repository: `mohamedamouseo-a11y/TCRM-MAIN-Tamiyouz-CRM-`
Pinned commit: `78273711727e834ca88029e39ff0f6ae302d427a`
Primary UI reference: `client/src/components/DeveloperHubTab.tsx`
Primary backend reference: `server/routes/developerHub.ts`

Use the pinned files as the source of truth for labels, information architecture, state model and Review → Execute flow. Do not replace the module with a generic Git form.

## Current TEA baseline

Persistent backend source:
- `/opt/TBA/openhands/automation/auth.py`
- `/opt/TBA/openhands/automation/config.py`
- `/opt/TBA/openhands/automation/developer_hub/router.py`

Frontend:
- `/opt/TBA/src/routes/developer-hub.tsx`
- `/opt/TBA/src/api/developer-hub-service.api.ts`
- `/opt/TBA/src/hooks/query/use-developer-hub.ts`
- `/opt/TBA/src/types/developer-hub.ts`
- `/opt/TBA/src/components/features/developer-hub/*`

Production API prefix:
`/api/automation/v1/developer-hub`

Do not patch `/root/.cache/uv/...`.

## Required GitHub module

### 1. GitHub account connection

Match TCRM behavior:
- GitHub Token input with show/hide
- Verify / Connect action
- Connected / Incomplete / Disconnected state
- verified GitHub login/account
- repository permission indicator
- Disconnect action
- token is write-only: never return raw token to browser after submission
- store token only through TEA's persistent server-side secure configuration/state mechanism

### 2. Repository selection

After successful connection:
- load accessible repositories
- show repository selector
- show private/public status where available
- show default branch and permission where available
- refresh repositories action
- save selected repository

### 3. Branch selection

For selected repository:
- load remote branches
- branch selector
- refresh branches action
- default/local branch awareness
- save selected branch
- do not silently switch repository/branch without explicit Super Admin action

### 4. Git synchronization controls

Match the TCRM layout/flow for:
- Push
- Pull
- Sync

Every mutation must use:
`Review / Preview → human Execute`

The Review result must show, where applicable:
- current branch
- selected repo + branch
- local ahead
- remote ahead
- dirty state
- sync state
- expected action
- changed files with direction/status
- file count
- conflicts
- blocked files/reasons
- Push Control/preflight result

The Execute action must be disabled until a successful, non-stale Review exists.

### 5. Review integrity

Bind each Review to the exact state it inspected using fingerprint / HEAD / selected repo / selected branch or equivalent.

If repository, branch, HEAD, working tree or relevant remote state changes after Review, Execute must reject the stale Review and require a new Review.

### 6. Operation UX

Match TCRM operation UX:
- review progress
- execute progress
- current step
- sanitized logs
- success/error/blocked states
- stable operation ID
- recover active operation after page reload where practical
- prevent duplicate execute on refresh/retry

### 7. Commit message

For Push/Sync where a commit is required:
- optional commit message field like TCRM
- sanitize/validate on backend
- no shell interpolation

### 8. Push Control integration

The existing Push Control section must be connected to the GitHub Review result rather than being decorative.

Display gates for:
- preflight
- tests
- build
- security
- remote sync

A mandatory failed gate blocks Execute.

## Backend capabilities

Extend the existing persistent TEA Developer Hub router only as required. Keep the API under:
`/api/automation/v1/developer-hub/*`

The final API should support the equivalent behavior of these TCRM capabilities:
- status
- GitHub auth/verify
- GitHub disconnect
- repository discovery
- branch discovery for a selected repository
- repository/branch selection persistence
- sync preview
- sync execute
- operation status/recovery

Reuse current TEA endpoint names where they already exist; do not create duplicate APIs just to copy TCRM path names.

## Security / operating policy

Non-negotiable:
- Super Admin only, enforced server-side on every endpoint
- non-Super-Admin authenticated user = 403
- anonymous = 401
- trusted TEA repository root only
- no arbitrary filesystem path from browser
- no arbitrary shell terminal/command field
- use argument arrays / safe subprocess APIs for Git
- validate repo URL, branch and commit message
- no force push
- no destructive reset/clean
- credentials never appear in responses, logs, audit, diff or source export
- serialize mutating Git operations
- audit success and failure
- no autonomous agent/background Git writes
- every Push/Pull/Sync Execute is initiated by an explicit human Super Admin action
- Developer Hub never restarts/deploys TEA

## Visual parity

The current screenshot is NOT sufficient parity because it only shows a GitHub status card.

The finished GitHub section must visibly provide the same interaction model as TCRM Main:
- connection panel
- token connect/verify state
- repository selector
- branch selector
- refresh/save/disconnect actions
- Push / Pull / Sync actions
- Review panel
- changed-files/conflicts/blockers result
- Execute action
- progress/log state

Use TEA/OpenHands native components/theme. Keep the existing dark UI style; do not import TCRM's component library wholesale.

## Scope protection

Do not modify:
- OpenHands dependencies
- React Router version
- nginx unless an existing Developer Hub API route is actually broken
- systemd
- ports
- normal conversation behavior
- unrelated Developer Hub sections (MCP, AI Context, Source Export, Audit) except wiring required for GitHub operation status/audit

Do not perform Git commit/push/pull/fetch/merge/rebase as the executor applying this patch.

Restart only NEW TEA services if required to load the code.

## Completion report

Return only:

```text
PATCH=TEA_DEVHUB_GITHUB_UI_TCRM_PARITY_V1
GITHUB_CONNECT_UI=PASS|FAIL
TOKEN_WRITE_ONLY=PASS|FAIL
REPOSITORY_SELECTOR=PASS|FAIL
BRANCH_SELECTOR=PASS|FAIL
DISCONNECT=PASS|FAIL
PUSH_REVIEW_EXECUTE=PASS|FAIL
PULL_REVIEW_EXECUTE=PASS|FAIL
SYNC_REVIEW_EXECUTE=PASS|FAIL
CHANGED_FILES_PREVIEW=PASS|FAIL
CONFLICT_BLOCKERS=PASS|FAIL
STALE_REVIEW_PROTECTION=PASS|FAIL
PUSH_CONTROL_WIRED=PASS|FAIL
OPERATION_PROGRESS=PASS|FAIL
OPERATION_RECOVERY=PASS|FAIL
SUPERADMIN_ONLY=PASS|FAIL
NON_SUPERADMIN_403=PASS|FAIL
DOMAIN_UI=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
PERSISTENT_SOURCE_ONLY=PASS|FAIL
CHANGED_FILES=<paths>
STATUS=READY|FAILED
```