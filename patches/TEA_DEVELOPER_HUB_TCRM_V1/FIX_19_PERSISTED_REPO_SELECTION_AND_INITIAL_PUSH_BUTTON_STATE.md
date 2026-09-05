# FIX 19 — Persisted repository selection + initial-push button state

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Accepted baseline

FIX 18 is the current code baseline, but its `STATUS=READY` is NOT sufficient because the real production screenshot after reload contradicts the report.

The intended selected repository remains:
`mohamedamouseo-a11y/TEA`

It is private and intentionally EMPTY.

## Confirmed production regression

The real browser screenshot after FIX 18 shows an inconsistent state:

- top `Repository status` card says repository = `https://github.com/mohamedamouseo-a11y/TEA`
- GitHub account is connected as `mohamedamouseo-a11y`
- Repository dropdown currently shows `mohamedamouseo-a11y/TOS-Patchs`
- Branch dropdown shows `Empty repository — no remote branches yet`
- `Save Selection` is disabled
- `Review Push` is disabled
- Synchronization Status still shows `Synced`

This proves the UI selection state is not bound correctly to the persisted selected repository/initial target branch after load/refresh/reload. The app is mixing persisted TEA status with a different transient dropdown repository.

The user must NOT have to reselect TEA after every reload, and an empty TEA repository must expose target branch `main` and enable `Review Push` after the persisted selection is valid.

Do NOT perform Initial Push in this task.

## Goal

Make the live UI and backend state consistent and deterministic:

1. persisted selected repository = `mohamedamouseo-a11y/TEA`
2. repository dropdown reflects the persisted repository after load/reload/refresh
3. empty remote exposes initial target branch `main`
4. saved empty-repository selection survives reload
5. `Review Push` becomes enabled for the valid persisted empty TEA selection
6. `Review Pull` / `Review Sync` stay blocked before first remote history
7. Synchronization Status must say `Remote Empty — Initial Push Required`, never `Synced`

## Inspect before editing

Inspect the actual live source/state and PROVE the cause:

- `/opt/TBA/openhands/automation/developer_hub/router.py`
- `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`
- `/opt/TBA/src/api/developer-hub-service.api.ts`
- `/opt/TBA/src/hooks/query/use-developer-hub.ts`
- `/opt/TBA/src/types/developer-hub.ts`
- persisted GitHub state file used by FIX 16

Specifically inspect:

- status response selected repository / selected branch fields
- repository-list fetch timing
- frontend `selectedRepo` initialization
- whether repo loading defaults to `repositories[0]`
- whether refresh overwrites persisted selection
- whether branch loading runs for the wrong transient repo
- whether empty-repo target branch `main` is stored separately from remote branch list
- button disabled predicates for Save Selection / Review Push
- sync-status calculation when persisted repo != transient dropdown repo

Do not guess from source names. Report the exact proven root cause.

## Required behavior

### 1. Persisted selection is authoritative

After status + repository list load:

- if persisted selected repo exists in the real GitHub repository list, set the dropdown to that exact `fullName`
- do NOT replace it with the first repository
- do NOT select a different repository merely because repository results arrive later
- refresh repositories must preserve the persisted/current valid selection
- changing the dropdown may create a local pending selection, but it must not change `Repository status` or synchronization truth until Save Selection succeeds

If the persisted selected repo is absent from the live GitHub list, show stale/invalid selection and require explicit human choice; do not silently pick another repository.

### 2. Separate saved selection from draft selection

If needed, maintain two explicit frontend concepts:

- saved/persisted repository+branch from backend status
- draft repository+branch currently chosen in selectors

The UI must make this state safe:

- `Save Selection` enabled only when draft differs from saved selection and draft is valid
- synchronization/review actions operate only on saved selection, never unsaved draft values
- after successful save, status reloads and saved state becomes the new source of truth

This prevents the exact screenshot condition where `Repository status` says TEA while selector says TOS-Patchs yet actions/status are ambiguous.

### 3. Empty repository target branch

For persisted `mohamedamouseo-a11y/TEA` with zero remote branches:

- show `Empty repository — no remote branches yet`
- also expose `Initial target branch: main` as the active draft/saved target branch
- `main` is a target branch, NOT a fabricated existing remote branch
- branch refresh must not erase the initial target branch
- reload must restore target branch `main` from persisted state

If the current selector cannot express this safely, add a minimal text/input or special target-branch control while preserving native TEA styling.

### 4. Save Selection rules

For empty remote TEA:

- selected repo = `mohamedamouseo-a11y/TEA`
- target branch = valid `main`
- Save Selection must succeed
- backend must persist both repo and target branch
- after reload, both must restore exactly

Do not require a remote branch to exist for empty repo selection.

### 5. Review Push enablement

For the SAVED empty TEA selection, Review Push is enabled when:

- token connected
- persisted repo = TEA
- persisted target branch valid
- remote empty = true
- no operation lock
- no unresolved stale draft selection

`Review Push` must generate a real preview with:

- expected action = `initial_push`
- repository = `mohamedamouseo-a11y/TEA`
- target branch = `main`
- remote branch exists = NO
- remote empty = YES
- real local HEAD
- real dirty state
- real file count/tree summary
- Push Control result
- stale-review fingerprint

### 6. Prevent actions on unsaved draft repo

If user changes Repository dropdown to another repo (e.g. TOS-Patchs) but has not saved it:

- Review Push/Pull/Sync must not silently run against the unsaved repo
- either disable review actions until Save Selection, or clearly keep them bound to saved TEA and label the unsaved draft
- preferred: disable actions with `Save repository selection first`

### 7. Truthful synchronization status

For saved TEA while remote empty:

`Synchronization Status = Remote Empty — Initial Push Required`

Never show `Synced` just because ahead/behind are numerically zero.

If the selector contains an unsaved different repo, synchronization status must still reflect SAVED TEA or explicitly show `Unsaved repository selection`; it must never merge data from two repos.

### 8. Reload acceptance proof

This fix is not READY until it survives a real browser reload.

Required exact sequence:

1. Select `mohamedamouseo-a11y/TEA`.
2. Target branch = `main`.
3. Save Selection.
4. Confirm backend persisted repo+branch.
5. Hard reload `/developer-hub`.
6. Confirm Repository dropdown still shows `mohamedamouseo-a11y/TEA`.
7. Confirm target branch still `main`.
8. Confirm status says `Remote Empty — Initial Push Required`.
9. Confirm Review Push is ACTIVE.
10. Click **Review Push only**.
11. Confirm initial-push preview is generated.
12. DO NOT execute.

Capture one screenshot after hard reload showing TEA selected + main target + Review Push enabled, and one screenshot of initial-push preview.

## Security / scope

Preserve all existing rules:

- Super Admin server-side gate
- persistent token storage
- real GitHub API repository discovery
- human-only Review -> Execute
- no force push
- no destructive reset/clean
- no token exposure/logging
- stale review protection
- Push Control
- normal OpenHands UI

Do NOT modify:

- nginx
- systemd definitions
- ports
- dependencies/React Router
- old TEAgent server
- unrelated Developer Hub cards

Executor applying this patch MUST NOT perform Git write/network workflow actions against TEA source:

- no git commit
- no git push
- no git pull
- no git fetch
- no merge/rebase/reset/clean
- no Initial Push execution

Read-only Git/GitHub diagnostics are allowed.

## Validation

Before restart:

- backend syntax/import PASS if backend changed
- frontend parse/type PASS
- frontend build PASS
- relevant tests PASS if present

Then production verification with real browser + hard reload as specified above.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_19_PERSISTED_REPO_SELECTION_AND_INITIAL_PUSH_BUTTON_STATE
ROOT_CAUSE=
PERSISTED_REPO_BEFORE=
DROPDOWN_REPO_BEFORE=
SELECTION_MISMATCH_CONFIRMED=YES|NO
PERSISTED_REPO_AFTER=
DROPDOWN_REPO_AFTER=
PERSISTED_BRANCH_AFTER=
REMOTE_BRANCH_COUNT=
REMOTE_EMPTY=YES|NO
DRAFT_SAVED_STATE_SEPARATED=PASS|FAIL|NOT_NEEDED
SAVE_SELECTION_EMPTY_REPO=PASS|FAIL
HARD_RELOAD_REPO_PERSISTENCE=PASS|FAIL
HARD_RELOAD_BRANCH_PERSISTENCE=PASS|FAIL
SYNC_STATUS_VALUE=
SYNC_STATUS_TRUTHFUL=PASS|FAIL
REVIEW_PUSH_ENABLED_AFTER_RELOAD=PASS|FAIL
REVIEW_PUSH_UNSAVED_DRAFT_GUARD=PASS|FAIL
INITIAL_PUSH_REVIEW=PASS|FAIL
INITIAL_PUSH_EXPECTED_ACTION=
INITIAL_PUSH_REPO=
INITIAL_PUSH_TARGET_BRANCH=
INITIAL_PUSH_FILE_COUNT=
INITIAL_PUSH_EXECUTED=NO
REVIEW_PULL_EMPTY_BLOCKED=PASS|FAIL
REVIEW_SYNC_EMPTY_BLOCKED=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL|NOT_CHANGED
TYPECHECK=PASS|FAIL
BUILD_VALIDATION=PASS|FAIL
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
SCREENSHOT_RELOAD_PATH=
SCREENSHOT_REVIEW_PATH=
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires the real browser hard-reload proof: TEA remains selected, target branch main remains selected, empty-remote status is truthful, Review Push is enabled, and Review Push produces an initial-push preview while `INITIAL_PUSH_EXECUTED=NO`.
