# FIX 14 — Real GitHub Repository Discovery + Empty Repository Support

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Baseline

FIX 13 is the current accepted UI baseline.

Expected live GitHub panel after FIX 13:
`/opt/TBA/src/components/features/developer-hub/github-panel.tsx`

Expected FIX 13 SHA256 from the real server report:
`b9e44b64e37e1bc9f46af343eb93d3a422258a3edb8511f5d2b47dc7ddbdf506`

Do not overwrite newer valid work blindly if the live SHA differs.

## Confirmed production problem

The production screenshot shows Developer Hub connected to GitHub but the selected repository is:
`TEA-Hub/openhands-tea`

The human Super Admin confirms the intended repository is named:
`TEA`

and that repository is currently EMPTY.

Therefore the repository connection/selection is not trustworthy yet. The current UI/backend must not display a hardcoded, placeholder, stale, or fabricated repository as if it were the real GitHub repository.

The fix must make repository discovery come from the authenticated GitHub token in real time and correctly support an empty destination repository.

## Source of truth

Match the current TCRM Developer Hub repository-discovery pattern from:

Repository:
`mohamedamouseo-a11y/TCRM-MAIN-Tamiyouz-CRM-`

Pinned commit:
`78273711727e834ca88029e39ff0f6ae302d427a`

Backend reference:
`server/routes/developerHub.ts`

Important reference behavior:
- verify token against GitHub
- load repositories from GitHub API, not static data
- use `/user/repos?per_page=100&page=N&sort=updated&affiliation=owner,collaborator,organization_member`
- include repositories where the token has usable permission
- expose normalized `fullName`, `repoUrl`, `private`, `defaultBranch`, `permission`
- load branches from the selected real repository

Reuse TEA's current persistent Developer Hub architecture and API prefix:
`/api/automation/v1/developer-hub/*`

Do not create another service or port.

## Task

### 1. Prove where the wrong repository comes from

Inspect the actual running TEA source/state before editing:
- `/opt/TBA/openhands/automation/developer_hub/router.py`
- persistent Developer Hub config/state used by that router
- `/opt/TBA/src/api/developer-hub-service.api.ts`
- `/opt/TBA/src/hooks/query/use-developer-hub.ts`
- `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`

Identify whether `TEA-Hub/openhands-tea` is coming from:
- hardcoded mock/default data
- stale persisted selection
- fake backend response
- frontend fallback
- incorrect repository discovery endpoint

Do not guess. Report the exact source.

### 2. Real repository discovery only

The GitHub repositories endpoint must query GitHub using the currently stored/verified token and return the REAL accessible repository list.

Requirements:
- no hardcoded repository names
- no placeholder repository objects
- no fake `HEAD`, ahead/behind, branch, sync, or permission values
- paginate the GitHub repositories API as needed
- request owner/collaborator/organization-member affiliations
- keep private repositories if the token can access them
- return normalized real metadata
- never return the raw token
- `Cache-Control: no-store`

If GitHub API fails, return a real error state. Do not silently substitute mock data.

### 3. Remove stale selected-repo state

If a persisted selected repository is not present in the current authenticated token's real repository list:
- mark the selection stale
- do not continue displaying it as valid
- clear/invalidate that repository selection safely
- require the Super Admin to select a real repository again

Do not automatically choose a different repository without explicit human selection.

### 4. The real `TEA` repository

After repository refresh, the dropdown must show the REAL repository named `TEA` if the authenticated token has access to it.

Do not hardcode `TEA-Hub/TEA`; use the exact `full_name` returned by GitHub. The owner may be an account or organization.

The UI must allow the Super Admin to select that repository and save the selection.

### 5. Correct empty-repository behavior

The repository `TEA` is intentionally empty right now.

An empty repository is NOT an error.

Support this state correctly:
- repository selection succeeds
- repository metadata is shown
- branches endpoint may legitimately return `[]`
- show a clear `Empty repository / no remote branches yet` state
- use GitHub's real `default_branch` when provided; otherwise allow the human to choose/confirm the initial branch name, default suggestion `main`
- do not fabricate a remote branch that does not exist

### 6. Initial Push review flow

For an empty selected remote repository, `Review Push` must support an INITIAL PUSH from the current local TEA Git branch.

Review must show real data:
- selected remote repository
- target branch
- current local branch
- local HEAD
- remote branch exists = NO
- remote is empty = YES
- exact changed/tracked files that would be included, or a truthful summary if the backend already models the initial tree differently
- Push Control gates
- expected action = `initial_push` or equivalent truthful existing enum/state

The review must NOT claim `Synced` before the initial push has actually happened.

### 7. Empty-remote action policy

Before the first push:
- `Review Push` = allowed when preconditions pass
- `Review Pull` = disabled/blocked with clear reason: remote has no branch/history
- `Review Sync` = disabled/blocked until remote has initial history, unless the existing architecture has a safe explicit initial-sync flow that is equivalent to initial push
- `Safe Cleanup` must not invent remote history

Every actual Git mutation remains:
`Review -> explicit HUMAN Super Admin Execute`

### 8. Repository Status must be real

The top `Repository status` card currently shows data such as repository URL, branch, HEAD, dirty, unpushed, ahead/behind and synced time.

After this fix:
- local repo data must come from the actual local `/opt/TBA` Git repository
- selected GitHub repository must come from real persistent selection
- remote comparison fields must reflect the selected real repository/branch
- if the selected remote is empty, show an explicit truthful empty/initial-push state
- never show `Synced` for an empty remote before initial push

### 9. Security / scope

Preserve:
- Super Admin only server-side
- anonymous = 401
- authenticated non-Super-Admin = 403
- token write-only / never returned to browser
- human-only Git mutations
- no force push
- no destructive reset/clean
- no arbitrary shell input
- no secret logging

Do NOT modify:
- nginx
- systemd
- ports
- React Router/dependencies
- old TEAgent server
- normal OpenHands conversation behavior

No new standalone service.

## Executor restrictions

The OpenHands executor applying FIX 14 must NOT itself perform Git write/network workflow operations against the TEA source repository:
- no git commit
- no git push
- no git pull
- no git fetch
- no merge/rebase/reset/clean

It may call the GitHub REST API read-only through the existing Developer Hub code/runtime to verify repository discovery, and may run local read-only Git commands such as `git status`, `git rev-parse`, `git ls-files`, and `git diff`.

Do not perform the first push during patch application. The human Super Admin will review and execute it later from Developer Hub.

## Validation

Before any restart:
- parse/type validation PASS
- build validation PASS
- backend syntax/import validation PASS
- run existing relevant tests if present

Restart only NEW TEA services if required.

Then verify through the production domain:

1. Disconnect/reconnect only if necessary to refresh the token-backed state; do not expose token.
2. Refresh repository list.
3. Prove repository list came from live GitHub response, not hardcoded state.
4. Confirm the real repository named `TEA` appears if token has access.
5. Select `TEA`.
6. Confirm empty repository state is represented honestly.
7. Confirm no fake `TEA-Hub/openhands-tea` remains unless GitHub actually returns that repository and the human explicitly selects it.
8. Confirm `Review Push` produces an INITIAL PUSH review and does not execute it.
9. Confirm Pull/Sync behavior is safely blocked/disabled for empty remote as specified.
10. Confirm normal OpenHands UI still works.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_14_REAL_GITHUB_REPO_DISCOVERY_EMPTY_REPO
ROOT_CAUSE=
WRONG_REPO_SOURCE=
HARDCODED_REPO_REMOVED=YES|NO|NOT_FOUND
STALE_SELECTION_HANDLED=PASS|FAIL
GITHUB_TOKEN_VERIFY=PASS|FAIL
REAL_REPO_API=PASS|FAIL
REAL_REPO_COUNT=
TEA_REPO_FULL_NAME=
TEA_REPO_VISIBLE=PASS|FAIL
TEA_REPO_SELECTED=PASS|FAIL
TEA_REPO_EMPTY=YES|NO
REMOTE_BRANCH_COUNT=
EMPTY_REPO_UI=PASS|FAIL
INITIAL_TARGET_BRANCH=
INITIAL_PUSH_REVIEW=PASS|FAIL
INITIAL_PUSH_EXECUTED=NO
PULL_EMPTY_REMOTE_BLOCKED=PASS|FAIL
SYNC_EMPTY_REMOTE_BLOCKED=PASS|FAIL
REPOSITORY_STATUS_TRUTHFUL=PASS|FAIL
FAKE_SYNC_STATE_REMOVED=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
PARSE_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL
BUILD_VALIDATION=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` is allowed only if repository discovery is live/token-backed, the real `TEA` repository is visible and selectable when accessible, empty-repository behavior is truthful, the initial push is reviewable but NOT executed by the patch executor, and no fabricated repository/sync data remains.