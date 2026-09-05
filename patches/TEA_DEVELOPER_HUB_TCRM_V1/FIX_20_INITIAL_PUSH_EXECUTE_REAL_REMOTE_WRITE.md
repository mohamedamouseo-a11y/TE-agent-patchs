# FIX 20 — Initial Push Execute must perform a real remote write

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed production discrepancy

The human Super Admin executed the reviewed `initial_push` action from Developer Hub and the UI showed:
- Progress 100%
- `Operation completed successfully`
- Operations card `SYNC success`

However independent GitHub verification of `mohamedamouseo-a11y/TEA` immediately after the click shows:
- repository `size=0`
- default branch metadata says `main`
- `GET /repos/mohamedamouseo-a11y/TEA/branches/main` returns 404 Branch not found

Therefore the UI reported success without creating the remote branch/history. This is a critical correctness bug. Do not trust current success state.

## Goal

Repair the existing Developer Hub Execute path so a human-approved `initial_push` review performs the actual Git write to the selected empty repository and success is reported ONLY after the remote branch exists and the pushed commit is verified remotely.

## Inspect first

Inspect the real live code/state and audit logs before editing:
- `/opt/TBA/openhands/automation/developer_hub/router.py`
- current operation registry/state used by Review/Execute
- current persisted GitHub repo/branch selection
- current audit/operation logs related to the last human Execute
- frontend only if the success state is also being fabricated client-side

Prove exactly what happened on the last Execute:
- which endpoint was called
- reviewed action / expected_action received
- whether initial_push reached the write branch
- whether a local commit was created or reused
- whether any remote push command/API call was attempted
- exit code/result of that write attempt
- why the operation was marked success despite the remote remaining empty

Never print the token.

## Required behavior

### 1. Real initial-push execution

For a valid non-stale reviewed action where:
- action = push
- expected_action = initial_push
- selected repo = `mohamedamouseo-a11y/TEA`
- target branch = `main`
- remote branch exists = NO
- remote repository empty = YES

then, only when the HUMAN Super Admin invokes Execute, the backend must perform the intended safe initial push using the existing secure Git environment/token handling.

No force push. No destructive reset/clean.

### 2. Success must be remote-verified

Do NOT mark Execute success just because a subprocess returned or a local step completed.

After the write attempt, verify via GitHub/read-only remote inspection that:
- remote branch `main` now exists
- remote HEAD SHA is present
- remote HEAD matches the commit that was intended to be pushed, or is otherwise proven to contain that exact reviewed tree
- remote repository is no longer empty

If any of those checks fail, operation status must be FAILED, not success.

### 3. Operation log must be truthful

The Progress/Operation Log and Operations card must reflect the real operation:
- action should be `PUSH` or `INITIAL PUSH`, not generic stale `SYNC`
- steps should reflect actual execution
- success only after remote verification
- failures include sanitized reason
- no token or credential leakage

Do not reuse a historical/stale operation entry as the current result.

### 4. Review integrity

Preserve:
- current review fingerprint/stale protection
- exact repo/branch binding
- Push Control gates
- human-only Execute
- Super Admin gate

If the review is stale after code/service restart, require a fresh Review Push before human Execute. Do not auto-execute anything.

### 5. Dirty local tree / commit handling

The reviewed initial-push preview reported:
- `fileCount=2206`
- dirty local tree = YES
- secret/safety audit = PASS

Do not silently exclude reviewed source files and do not include secrets/runtime artifacts outside the reviewed tree.

Use the current controlled-push policy for commit creation. If a commit is required, use the optional human commit message or a safe default only if current architecture already defines one.

### 6. Executor restrictions for applying FIX 20

The OpenHands patch executor MUST NOT itself perform the actual initial push.

During patch application it may:
- inspect source/state/logs
- run parse/type/build/backend validation
- use read-only local Git commands
- use read-only GitHub API calls

It must NOT:
- git push
- git pull
- git fetch
- git commit
- merge/rebase/reset/clean
- call the Developer Hub Execute endpoint

After code repair, the human Super Admin will run Review Push again and explicitly Execute.

## Validation before handoff

Before any restart:
- backend syntax/import validation PASS
- frontend type/build PASS if changed
- tests PASS where available

After restart only if required:
- token persistence still works
- selected repo remains `mohamedamouseo-a11y/TEA`
- remote still empty before human action
- Review Push can produce a fresh `initial_push` review
- Execute remains human-only
- normal OpenHands UI works

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_20_INITIAL_PUSH_EXECUTE_REAL_REMOTE_WRITE
LAST_EXECUTE_ENDPOINT=
LAST_EXECUTE_EXPECTED_ACTION=
LAST_EXECUTE_WRITE_ATTEMPTED=YES|NO|UNKNOWN
LAST_EXECUTE_WRITE_RESULT=
LAST_EXECUTE_MARKED_SUCCESS=YES|NO
REMOTE_BEFORE_EMPTY=YES|NO
REMOTE_BEFORE_MAIN_EXISTS=YES|NO
ROOT_CAUSE=
EXECUTE_INITIAL_PUSH_PATH_REPAIRED=PASS|FAIL
REMOTE_VERIFICATION_REQUIRED_FOR_SUCCESS=PASS|FAIL
OPERATION_LOG_TRUTHFUL=PASS|FAIL
STALE_OPERATION_RESULT_FIXED=PASS|FAIL
REVIEW_FINGERPRINT_PRESERVED=PASS|FAIL
PUSH_CONTROL_PRESERVED=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
TOKEN_SECRET_SAFE=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL|NOT_CHANGED
BUILD_VALIDATION=PASS|FAIL|NOT_CHANGED
FRESH_INITIAL_PUSH_REVIEW_AVAILABLE=PASS|FAIL
INITIAL_PUSH_EXECUTED_BY_PATCH=NO
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` means the execution code path is repaired and ready for a NEW human Review Push -> Execute cycle. It does NOT mean the initial push itself has happened.
