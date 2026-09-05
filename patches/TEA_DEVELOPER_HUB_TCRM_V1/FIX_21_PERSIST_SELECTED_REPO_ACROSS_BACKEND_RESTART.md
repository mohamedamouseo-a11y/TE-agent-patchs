# FIX 21 — Persist selected GitHub repository/branch across backend restart

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed current state

FIX 20 repaired the real initial-push execute path, but its validation found a separate persistence regression after service restart:

- before restart, the intended saved repository was `mohamedamouseo-a11y/TEA`
- after restart, the running Developer Hub reverted to `TOS-Patchs`
- because the wrong repository became authoritative, fresh Review Push resolved to `sync` instead of `initial_push`
- patch executor did NOT execute any Git write

This means repository/branch persistence is still not restart-safe even though frontend hard-reload persistence was fixed previously.

## Goal

Make the selected GitHub repository and selected branch/initial target branch durable on the backend across TEA automation/backend restarts.

The authoritative saved state after restart must remain:
- repository: `mohamedamouseo-a11y/TEA`
- branch / initial target branch: `main`
- remote empty: YES
- expected next reviewed action: `initial_push`

Do not hardcode TEA as a product default. Persist whatever the human Super Admin explicitly saves. For this verification, TEA is the currently saved real selection.

## Inspect first

Inspect the actual live implementation and state before editing:
- `/opt/TBA/openhands/automation/developer_hub/router.py`
- `/root/.openhands/tba/github-state.json`
- any other Developer Hub persisted selection file actually used at runtime
- frontend GitHub panel only if backend output is already correct and UI alone is wrong

Prove exactly:
- where Save Selection writes repository + branch
- what structure is stored on disk
- how module startup loads that state
- whether startup overwrites persisted selection with a default/first repository
- whether multiple worker/process states disagree

## Required behavior

1. `Save Selection` must persist repository + branch server-side atomically.
2. Service/backend restart must reload the same saved repository + branch.
3. Startup must never replace a valid saved repository with `repos[0]` or another default.
4. If the saved repository is not accessible to the current token, mark it stale and require human reselection. Do not silently substitute another repository.
5. For empty repository `mohamedamouseo-a11y/TEA`, persist `main` as the initial target branch even though no remote branch exists yet.
6. After restart, production UI must show TEA + main without manual reselection.
7. Synchronization status must remain `Remote Empty — Initial Push Required` until the first real push succeeds.
8. Fresh Review Push after restart must return `expected_action=initial_push` for TEA.
9. Do NOT execute the push in this patch task.

## Preserve

Preserve FIX 20 real execute implementation:
- real Git write path
- remote verification required before success
- truthful operation type/log/audit
- stale-review protection
- Push Control
- Super Admin gate
- token persistence/security

## Scope protection

Do NOT modify nginx, systemd unit definitions, ports, dependencies, old TEAgent server, or normal OpenHands behavior.

Executor restrictions:
- no git commit/push/pull/fetch/merge/rebase/reset/clean against TEA source
- no Execute Reviewed Action
- no Initial Push
- read-only Git/GitHub diagnostics allowed

## Validation

After the minimum fix:
1. Save `mohamedamouseo-a11y/TEA` + `main` using the normal Developer Hub persistence path if not already saved.
2. Confirm persisted file contains the normalized repository selection and branch metadata without secrets.
3. Restart only the NEW TEA automation/backend service.
4. Without any manual reselection, verify backend status returns TEA + main.
5. Verify production dropdown shows TEA + main.
6. Verify remote still empty and `main` does not exist remotely.
7. Verify status says `Remote Empty — Initial Push Required`.
8. Run Review Push only and confirm `expected_action=initial_push`.
9. DO NOT execute.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_21_PERSIST_SELECTED_REPO_ACROSS_BACKEND_RESTART
ROOT_CAUSE=
SELECTION_STORAGE_PATH=
SELECTION_PERSISTED_BEFORE_RESTART=PASS|FAIL
PERSISTED_REPO_BEFORE=
PERSISTED_BRANCH_BEFORE=
BACKEND_RESTART=PASS|FAIL
PERSISTED_REPO_AFTER=
PERSISTED_BRANCH_AFTER=
BACKEND_REPO_AFTER=
BACKEND_BRANCH_AFTER=
UI_REPO_AFTER=PASS|FAIL
UI_BRANCH_AFTER=PASS|FAIL
WRONG_DEFAULT_SUBSTITUTION_REMOVED=PASS|FAIL
REMOTE_EMPTY_AFTER=YES|NO
REMOTE_MAIN_EXISTS_AFTER=YES|NO
SYNC_STATUS_VALUE=
SYNC_STATUS_TRUTHFUL=PASS|FAIL
FRESH_INITIAL_PUSH_REVIEW_AVAILABLE=PASS|FAIL
INITIAL_PUSH_EXPECTED_ACTION=
INITIAL_PUSH_EXECUTED=NO
FIX20_REAL_EXECUTE_PATH_PRESERVED=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
TOKEN_SECRET_SAFE=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL|NOT_CHANGED
BUILD_VALIDATION=PASS|FAIL|NOT_CHANGED
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires the saved selection to survive a real backend restart and a fresh initial-push review to be available without manual reselection, while no push is executed.
