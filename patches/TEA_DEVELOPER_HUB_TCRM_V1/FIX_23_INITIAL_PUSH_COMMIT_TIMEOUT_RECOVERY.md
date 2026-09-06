# FIX 23 — Initial push local commit timeout recovery

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed production failure

After FIX 20/21/22, the human Super Admin created a fresh reviewed `initial_push` and clicked **Execute Reviewed Action**.

The production UI shows a truthful FAILED operation at 20% with these real steps:

- `Starting operation...`
- `Staging all tracked files...`
- `Files staged successfully.`
- `Creating commit: Initial push to main`
- `FAILED: Command ['git', 'commit', '-m', 'Initial push to main', '--allow-empty'] timed out after 30 seconds`

The reviewed tree contains about 2233 files. The failure happened before any confirmed remote push.

This is a new execution-path issue. Do NOT retry Execute blindly and do NOT assume the reason is GPG, hooks, or repository size until proven.

## Goal

Diagnose the exact reason the local `git commit` exceeded the hard 30-second subprocess timeout, repair the commit execution path safely, and leave the system ready for a **new human Review Push -> Execute** cycle.

The patch executor MUST NOT commit or push anything itself.

## Inspect first — no writes

Inspect only what is required:

- `/opt/TBA/openhands/automation/developer_hub/router.py`
- helper(s) used by `execute_preview` to run git subprocesses
- current operation/audit entry for the failed initial push
- read-only local Git state under `/opt/TBA`

Record sanitized evidence for:

1. Exact timeout value used for `git commit` and where it is defined.
2. Whether the timed-out process left a local commit behind:
   - current `HEAD`
   - whether this repository has any local commit/history
   - whether HEAD changed compared with the pre-execute review/operation metadata
3. Current index state after the failed execute:
   - count of staged files
   - count of unstaged/untracked files
   - do NOT reset/unstage/clean anything
4. Commit signing configuration, sanitized only:
   - `commit.gpgsign`
   - signing format if configured
   - whether signing is enabled; never print keys/secrets
5. Hooks:
   - `core.hooksPath`
   - names of executable commit-related hooks only (`pre-commit`, `prepare-commit-msg`, `commit-msg`, `post-commit`, etc.)
   - do not print secret-bearing hook contents
6. Git identity readiness:
   - whether author name/email resolve successfully using read-only Git identity commands
   - do not change global/system config during diagnosis
7. Determine whether any timed-out child process remains running from the failed Execute.

Classify the primary cause using evidence, for example:

- `COMMIT_TIMEOUT_TOO_LOW`
- `COMMIT_HOOK_EXCEEDED_TIMEOUT`
- `GPG_SIGNING_INTERACTIVE_BLOCK`
- `GIT_IDENTITY_BLOCK`
- `PROCESS_TERMINATION_BUG`
- another exact proven cause

Do not guess.

## Repair requirements

### 1. Operation-specific timeout

A 30-second generic subprocess timeout must not make a legitimate first commit of a ~2200-file reviewed tree fail prematurely.

Use an operation-specific commit timeout large enough for this workload (recommended 180–300 seconds), while keeping bounded execution.

Do NOT remove timeouts entirely.

Remote push may have its own separately bounded timeout appropriate for network operations.

### 2. Non-interactive execution

The server-side human-approved Execute path must never wait on an interactive terminal prompt.

For git subprocesses involved in commit/push, use safe non-interactive environment controls where applicable, such as:

- `GIT_TERMINAL_PROMPT=0`
- non-interactive editor behavior when a message is already supplied

If and only if diagnosis proves configured GPG signing is causing an interactive block, make the Developer Hub-created operational commit explicitly non-interactive without exposing keys or weakening unrelated repository policy. Do NOT globally rewrite the user's Git configuration.

### 3. Do not bypass hooks blindly

Do NOT add `--no-verify` merely to make the commit finish.

If a legitimate hook is the reason, preserve it and give the commit an appropriate bounded timeout unless the existing controlled Push Control already performs the exact same check and there is a strong, documented reason to avoid duplicate execution.

Any hook bypass requires explicit evidence and must not silently weaken safeguards.

### 4. Partial-failure recovery

The failed Execute already staged files. The repair must correctly handle all possible current states without destructive cleanup:

- staged tree exists, no new commit exists
- commit actually exists despite timeout (for example a post-commit hook stalled)
- mixed staged/unstaged state

Do NOT run reset, clean, checkout, restore, or discard user work.

Before a future Execute:

- stale-review/fingerprint protection must re-evaluate the current state
- if a commit already exists, do not create a duplicate initial commit blindly
- if no commit exists, a fresh reviewed action may create the intended commit
- exact repo/branch binding must remain TEA/main

### 5. Process timeout handling

When a subprocess exceeds its bound:

- terminate the whole spawned process group safely so hooks/children cannot remain orphaned
- capture a sanitized timeout reason
- mark operation FAILED
- never continue to push after a failed/uncertain commit step

### 6. Success rules remain strict

Preserve FIX 20:

- actual push only on human Execute
- no force push
- remote verification required before marking success
- `main` must exist remotely after success
- remote HEAD/tree must correspond to the intended reviewed commit/tree
- no fabricated success

### 7. UI/log truthfulness

Preserve the current truthful failure behavior.

Progress/Operation Log must identify the real stage (`commit`, `push`, or `remote verification`) and sanitized failure reason.

Do not show stale historical `SYNC success` as the current operation.

## Security / scope

Preserve:

- FIX 16 token persistence/security
- FIX 17 real GitHub discovery
- FIX 19/21 TEA + main persistence
- FIX 20 real execute + remote verification
- FIX 22 frontend provider repair
- Super Admin gate
- Push Control
- review fingerprint/stale protection
- secret scan behavior
- normal OpenHands UI

Do NOT modify nginx, systemd unit definitions, ports, unrelated frontend, dependencies, or old TEAgent server.

## Patch executor restrictions

During FIX 23 application:

- NO `git commit`
- NO `git push`
- NO `git pull`
- NO `git fetch`
- NO merge/rebase/reset/clean/checkout/restore
- NO Developer Hub Execute endpoint
- NO Initial Push
- NO token/secret output

Read-only Git/GitHub diagnostics are allowed.

Do not alter the index merely to validate the fix.

## Validation before handoff

1. Backend syntax/import validation PASS.
2. Frontend type/build only if frontend changed.
3. Confirm no orphaned git/commit process remains.
4. Confirm selected repository still `mohamedamouseo-a11y/TEA` and target `main`.
5. Confirm GitHub remote is still empty / `main` absent unless independent read-only evidence proves the prior failed command somehow completed remotely.
6. Confirm repaired Execute code uses bounded commit timeout >30 seconds and non-interactive process controls.
7. Confirm a commit failure/timeout cannot fall through to push.
8. Confirm remote verification remains mandatory for success.
9. Generate a **fresh Review Push only** if possible without Git writes, and report whether it is valid or blocked/stale because of the staged state.
10. Do NOT Execute.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_23_INITIAL_PUSH_COMMIT_TIMEOUT_RECOVERY
FAILED_COMMAND=
OLD_COMMIT_TIMEOUT_SECONDS=
ROOT_CAUSE_CLASS=
ROOT_CAUSE=
COMMIT_GPGSIGN=
COMMIT_HOOKS=
GIT_IDENTITY_READY=YES|NO
ORPHAN_GIT_PROCESS_FOUND=YES|NO
LOCAL_HEAD_BEFORE_RECOVERY=
LOCAL_COMMIT_CREATED_BY_FAILED_EXECUTE=YES|NO|UNKNOWN
STAGED_FILE_COUNT_AFTER_FAILURE=
UNSTAGED_FILE_COUNT_AFTER_FAILURE=
UNTRACKED_FILE_COUNT_AFTER_FAILURE=
REMOTE_EMPTY=YES|NO
REMOTE_MAIN_EXISTS=YES|NO
NEW_COMMIT_TIMEOUT_SECONDS=
NONINTERACTIVE_GIT_ENV=PASS|FAIL
PROCESS_GROUP_TIMEOUT_CLEANUP=PASS|FAIL
HOOK_SAFETY_PRESERVED=PASS|FAIL
DUPLICATE_COMMIT_GUARD=PASS|FAIL
COMMIT_FAILURE_BLOCKS_PUSH=PASS|FAIL
REMOTE_VERIFICATION_REQUIRED_FOR_SUCCESS=PASS|FAIL
REVIEW_FINGERPRINT_PRESERVED=PASS|FAIL
PUSH_CONTROL_PRESERVED=PASS|FAIL
FRESH_INITIAL_PUSH_REVIEW=PASS|BLOCKED|FAIL
FRESH_REVIEW_EXPECTED_ACTION=
INITIAL_PUSH_EXECUTED_BY_PATCH=NO
SUPERADMIN_GATE=PASS|FAIL
TOKEN_SECRET_SAFE=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL|NOT_CHANGED
BUILD_VALIDATION=PASS|FAIL|NOT_CHANGED
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=
STATUS=READY|NEEDS_HUMAN_REVIEW|FAILED
```

`STATUS=READY` means the commit timeout/execution path is repaired and a fresh human review can safely proceed. It does NOT mean the initial push has been executed.
