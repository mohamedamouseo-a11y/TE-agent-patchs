# FIX 18 — Empty repository initial branch + Review Push enablement

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed production state

The real GitHub repository is now correctly discovered and selected:

- account: `mohamedamouseo-a11y`
- repository: `mohamedamouseo-a11y/TEA`
- repository is private
- repository is confirmed EMPTY
- GitHub token persists across restart
- backend and UI repository discovery both work

The production screenshot after selection proves one remaining empty-repository flow bug:

- Repository shows `mohamedamouseo-a11y/TEA (private)`
- Branch shows `No branches`
- `Save Selection` is disabled
- `Review Push` is disabled
- `Review Pull` is disabled
- `Review Sync` is disabled
- Synchronization Status incorrectly shows `Synced`

For an empty remote repository this is wrong. There is no remote history yet, so TEA must expose an initial target branch (suggest `main`) and allow a HUMAN Super Admin to run **Review Push** for the initial upload.

Do NOT perform the initial push in this patch task.

## Goal

Make the empty repository state truthful and usable:

1. allow the real selected empty repository to be saved even though it has zero remote branches
2. expose an initial target branch, default suggestion `main`
3. allow `Review Push` for the initial push
4. keep `Review Pull` and `Review Sync` blocked until remote history exists
5. do not report `Synced` before the first push
6. preserve all security and explicit human Review -> Execute controls

## Inspect first

Inspect the actual live code/state before editing:

- `/opt/TBA/openhands/automation/developer_hub/router.py`
- `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`
- `/opt/TBA/src/api/developer-hub-service.api.ts`
- `/opt/TBA/src/hooks/query/use-developer-hub.ts`
- `/opt/TBA/src/types/developer-hub.ts`

Prove exactly why empty repos currently disable Save Selection and Review Push.

Possible causes include, but must be proven:
- frontend requires `selectedBranch` to exist in remote branch list
- backend save-selection validation rejects a branch not yet present remotely
- review-push endpoint requires an existing remote branch
- status logic assumes ahead=0/behind=0 means synced even when remote branch/history does not exist

## Required behavior

### 1. Empty repo branch UX

When the selected repository has zero remote branches:

- show explicit text/state: `Empty repository — no remote branches yet`
- provide an editable/confirmable initial target branch field or selector value
- default suggestion: `main`
- do NOT fabricate a remote branch; distinguish `target branch` from `existing remote branch`
- user must be able to save the repository + initial target branch selection

If the current UI architecture only supports a branch selector, minimally adapt it so an empty-repo special option/input can represent `main` as an **initial target branch**, not as an existing remote branch.

### 2. Save Selection

For an empty repository:

- Save Selection must be enabled when a valid initial target branch name is present
- backend must persist selected repository and target branch safely
- validation must allow a target branch that does not yet exist remotely only when the selected remote is proven empty
- preserve existing branch-name validation and Super Admin gate

### 3. Truthful synchronization status

Before first push, never show `Synced`.

Use a truthful state such as:
- `Initial push required`
- `Remote empty`
- or equivalent current enum/state

Repository Status / Synchronization Status should truthfully represent:
- local branch: current local branch (currently `main`)
- local HEAD: real current HEAD
- selected remote repo: `mohamedamouseo-a11y/TEA`
- target branch: `main` (or explicit human-selected branch)
- remote branch exists: NO
- remote empty: YES
- ahead/behind should not be presented as proof of sync when no remote branch exists

### 4. Review Push for initial push

Enable **Review Push** only after:
- repository selected
- valid initial target branch saved/confirmed
- token is connected
- remote repository is proven empty
- no Git operation lock is active

Review Push must be real, not decorative.

The review result must include truthful data:
- action = push
- expected action = `initial_push` or equivalent truthful value
- repository = `mohamedamouseo-a11y/TEA`
- target branch
- current local branch
- local HEAD
- `remoteBranchExists=false`
- `remoteEmpty=true`
- dirty state
- exact tracked/changed file summary to be uploaded, or the current architecture's truthful complete-tree summary
- Push Control/preflight gates
- blockers/conflicts if any
- review fingerprint/stale protection

Do not require an existing remote ref to build this review.

### 5. Initial push execution path must exist but MUST NOT be executed now

The human UI may prepare a valid Execute action after successful Review Push.

The patch executor MUST NOT click or call Execute and MUST NOT push to GitHub.

Execution implementation must preserve:
- explicit HUMAN Super Admin action
- no force push
- safe remote URL/token handling
- target branch validation
- stale review rejection
- Push Control gates
- no secrets in logs

If execution code currently cannot handle a missing remote branch, fix only the code path required so a later human Execute can create the initial branch safely.

### 6. Pull / Sync policy before first push

For an empty remote:

- Review Pull = disabled or blocked with clear reason: remote has no history/branch
- Review Sync = disabled or blocked with clear reason: initial push required first
- Safe Cleanup must not invent remote history and must not be used as a substitute for initial push

### 7. Dirty local tree

The screenshot shows `Dirty (22)`.

Do NOT hide or clear this.

Review Push must truthfully explain what will be included or blocked under the existing controlled-push policy. Do not automatically commit, clean, reset, or discard anything during patch application.

## Security / scope

Preserve:
- Super Admin server-side authorization
- persistent token storage from FIX 16
- real repository discovery from FIX 17
- full Developer Hub UI from FIX 13
- Review -> explicit human Execute policy
- stale-review protection
- Push Control gates
- normal OpenHands UI

Do NOT modify:
- nginx
- systemd unit definitions
- ports
- dependencies / React Router
- old TEAgent server
- unrelated Developer Hub cards

No new service or database.

Executor restrictions:
- no git commit
- no git push
- no git pull
- no git fetch
- no merge/rebase/reset/clean
- no Initial Push execution
- read-only Git commands are allowed
- read-only GitHub API calls are allowed

## Validation

Before restart:
- backend syntax/import validation PASS
- frontend parse/type validation PASS if changed
- frontend build PASS if changed
- relevant tests PASS if available

After restart only if needed, verify production:

1. `mohamedamouseo-a11y/TEA` remains selected/available
2. empty remote is clearly identified
3. initial target branch `main` is available/selected
4. Save Selection works for the empty repo
5. Synchronization Status is NOT `Synced`; it says initial push required/remote empty
6. Review Push is enabled
7. Review Pull remains blocked/disabled
8. Review Sync remains blocked/disabled
9. clicking **Review Push only** produces a real initial-push preview
10. DO NOT execute the reviewed action
11. normal OpenHands UI still works

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_18_EMPTY_REPO_INITIAL_BRANCH_AND_REVIEW_PUSH
ROOT_CAUSE=
TEA_REPO_SELECTED=PASS|FAIL
REMOTE_EMPTY=YES|NO
REMOTE_BRANCH_COUNT=
INITIAL_TARGET_BRANCH=
INITIAL_BRANCH_UI=PASS|FAIL
SAVE_SELECTION_EMPTY_REPO=PASS|FAIL
SYNC_STATUS_TRUTHFUL=PASS|FAIL
SYNC_STATUS_VALUE=
REVIEW_PUSH_ENABLED=PASS|FAIL
INITIAL_PUSH_REVIEW=PASS|FAIL
INITIAL_PUSH_EXPECTED_ACTION=
REMOTE_BRANCH_EXISTS=YES|NO
REVIEW_FILE_COUNT=
REVIEW_DIRTY_STATE=
PUSH_CONTROL_RESULT=PASS|FAIL|BLOCKED
STALE_REVIEW_PROTECTION=PASS|FAIL
REVIEW_PULL_EMPTY_BLOCKED=PASS|FAIL
REVIEW_SYNC_EMPTY_BLOCKED=PASS|FAIL
INITIAL_PUSH_EXECUTED=NO
SUPERADMIN_GATE=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL|NOT_CHANGED
BUILD_VALIDATION=PASS|FAIL|NOT_CHANGED
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires a real production **Review Push** preview for the empty `TEA` repository while the actual initial push remains unexecuted.
