# FIX 25 — Initial Push from detached isolated worktree must use a fully-qualified destination ref

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed production failure

After FIX 24, the human Super Admin executed a fresh `initial_push` review.

The isolated execution path worked through commit creation:

- isolated worktree was created under `/tmp/tea-push-*`
- staging completed
- commit completed successfully
- local commit SHA was created (`af460a2b7cd7` in the observed run)

The push step then failed with Git's refspec error:

`The destination you provided is not a full refname`

The production log shows the operation was pushing detached `HEAD` toward target branch `main` in the empty repository.

Independent GitHub verification still shows:

- `mohamedamouseo-a11y/TEA` size = 0
- remote branch `main` does not exist

Therefore the failure is specifically the initial-push refspec used from the detached isolated worktree. For an empty remote, Git cannot infer a destination from an abbreviated `HEAD:main`-style refspec. The destination must be explicit and fully qualified.

## Goal

Repair only the real initial-push remote refspec so a HUMAN-approved initial push from the isolated detached worktree uses an explicit destination branch ref, e.g.:

`HEAD:refs/heads/main`

Do not execute the push while applying this patch.

## Inspect first

Inspect the actual implementation in:

- `/opt/TBA/openhands/automation/developer_hub/router.py`

Locate the exact command used for the failed push and prove:

- source ref used
- destination ref used
- cwd is the isolated worktree
- target branch is the persisted/reviewed branch (`main` in this case)
- why Git rejected the abbreviated destination on an empty remote

Do not print the token or authenticated remote URL.

## Required repair

1. For `expected_action=initial_push` from the detached isolated workspace, use a fully-qualified remote branch destination:
   - source: `HEAD`
   - destination: `refs/heads/<validated_target_branch>`
   - effective refspec: `HEAD:refs/heads/<validated_target_branch>`

2. Continue to use the exact repository + target branch bound to the non-stale review.

3. Validate branch name before building the refspec. Do not permit arbitrary ref injection.

4. Do NOT use force push.

5. Do NOT silently fall back to another branch.

6. Preserve the secure token/remote handling. Never expose credentials in process logs, audit, errors, or UI.

7. Preserve FIX 24 isolated-worktree execution:
   - no staging/commit in live `/opt/TBA`
   - hooks run in isolated workspace
   - temp workspace cleanup

8. Preserve FIX 23:
   - 240-second commit timeout
   - non-interactive environment
   - process-group timeout cleanup
   - hook safety
   - duplicate-commit protection

9. Preserve FIX 20:
   - remote verification is mandatory before success
   - remote branch must exist after push
   - remote HEAD/tree must match intended reviewed commit/tree
   - operation must be FAILED if remote verification fails

10. Operation log must show a sanitized truthful sequence, for example:
   - creating isolated worktree
   - staging reviewed tree
   - creating commit
   - pushing HEAD to `refs/heads/main`
   - verifying remote branch/HEAD
   - success only after verification

## Important failed-attempt state

The previous human Execute created a commit in a temporary isolated worktree but the push failed. The temporary worktree should have been cleaned up by FIX 24.

Do not assume that failed isolated commit is reusable. On the next human cycle:

- require a fresh Review Push
- create/reuse commit only according to the existing duplicate/stale safeguards
- never push an old unbound commit from a failed review

## Executor restrictions

While applying FIX 25, OpenHands MUST NOT:

- call Execute Reviewed Action
- run `git push`
- run `git commit`
- run `git pull` / `git fetch`
- merge/rebase/reset/clean live TEA source
- create the remote branch

Read-only Git/GitHub diagnostics are allowed.

## Validation before handoff

Verify without executing a push:

1. backend syntax/import validation PASS
2. production `/` and `/developer-hub` still work
3. persisted repo remains `mohamedamouseo-a11y/TEA`
4. persisted target branch remains `main`
5. remote remains empty
6. remote `main` remains absent
7. fresh Review Push returns `expected_action=initial_push`
8. reviewed execute command/path is proven to build `HEAD:refs/heads/main` (or equivalent explicit full destination ref)
9. live `/opt/TBA` is not used as commit cwd
10. no push is executed by patch executor

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_25_INITIAL_PUSH_FULLY_QUALIFIED_REFSPEC
FAILED_PUSH_SOURCE_REF=
FAILED_PUSH_DEST_REF=
ROOT_CAUSE=
INITIAL_PUSH_REFSPEC_AFTER=
DESTINATION_FULLY_QUALIFIED=PASS|FAIL
TARGET_BRANCH_VALIDATION=PASS|FAIL
FORCE_PUSH_USED=NO
ISOLATED_WORKSPACE_PRESERVED=PASS|FAIL
LIVE_TBA_USED_AS_COMMIT_CWD=NO
FIX23_TIMEOUT_HOOK_SAFETY_PRESERVED=PASS|FAIL
FIX20_REMOTE_VERIFICATION_PRESERVED=PASS|FAIL
TOKEN_SECRET_SAFE=PASS|FAIL
REMOTE_EMPTY=YES|NO
REMOTE_MAIN_EXISTS=YES|NO
PERSISTED_REPO=
PERSISTED_BRANCH=
FRESH_INITIAL_PUSH_REVIEW=PASS|FAIL
FRESH_REVIEW_EXPECTED_ACTION=
INITIAL_PUSH_EXECUTED_BY_PATCH=NO
BACKEND_VALIDATION=PASS|FAIL
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires the initial-push command path to use a validated fully-qualified destination ref and a fresh human `initial_push` review to be available, while GitHub remains untouched by the patch executor.
