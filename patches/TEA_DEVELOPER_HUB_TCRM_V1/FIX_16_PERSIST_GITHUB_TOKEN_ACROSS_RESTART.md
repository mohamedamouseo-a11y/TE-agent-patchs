# FIX 16 — Persist GitHub token across TEA restarts

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed root cause from FIX 15

FIX 15 proved this is NOT yet a GitHub permission diagnosis.

The actual current problem is:

- GitHub token is held only in module-level memory (`_github_token`) inside the Developer Hub backend.
- Restarting the TEA automation/backend process clears that variable.
- After restart, Developer Hub reports disconnected / `token_set=false` and repository discovery returns zero because there is no token left to call GitHub with.
- Therefore `No repositories` after restart is currently caused by token loss, not proven PAT permission failure.

The previous `STATUS=NEEDS_TOKEN_PERMISSION_FIX` must not be treated as authoritative until token persistence is fixed and the GitHub API is tested again with the surviving token.

## Goal

Make the GitHub connection persistent server-side so the Super Admin enters the token once, then TEA can still verify GitHub and discover repositories after service restart without exposing the raw token to the browser or logs.

After this fix, run the real repository discovery again. Only then classify PAT permissions if GitHub itself returns zero repositories.

## Scope

Primary live backend:
`/opt/TBA/openhands/automation/developer_hub/router.py`

Also inspect the current OpenHands automation/config/auth source under `/opt/TBA/openhands/automation/` to reuse an EXISTING persistent configuration / secret storage mechanism if available.

Do not create a new service or port.

## Task

### 1. Inspect current token lifecycle first

Before editing, prove the current implementation:

- where `_github_token` is declared
- how Connect stores it
- how Disconnect clears it
- how status determines `token_set` / connected state
- how repository discovery reads the token
- what persistent state/config mechanism Developer Hub already uses for repository selection or other settings

Return the exact current token source in `OLD_TOKEN_STORAGE=`.

### 2. Replace memory-only token storage

Remove the token's dependency on module-level memory as the source of truth.

Required behavior:

- Connect verifies the submitted PAT against GitHub first.
- Only after successful verification, persist the token server-side.
- Repository discovery, branch discovery, Review, and Execute read the token from persistent server-side storage.
- Status may return only boolean/masked metadata such as `token_set`, login, verified time; NEVER the raw token.
- Disconnect deletes/invalidates the persisted token and clears associated connection metadata safely.
- A process/service restart must NOT disconnect GitHub.

### 3. Use secure persistent storage

Preferred order:

1. Reuse any existing OpenHands/TEA secret or encrypted configuration mechanism already available in `/opt/TBA`.
2. If no existing encrypted secret mechanism exists, use a dedicated server-side secret file outside frontend/public assets and outside Git-tracked source, owned by root/service user with mode `0600`, with atomic write/replace.

Do NOT:

- store PAT in frontend localStorage/sessionStorage/cookies
- return PAT in API responses
- put PAT in Git-tracked files
- put PAT in normal audit logs
- print PAT to terminal/report
- write PAT into source code
- commit PAT into Git

If a persistent JSON state file already exists, do NOT store the raw token inside a broadly readable JSON file unless it is protected as a dedicated secret with strict file permissions and never exported.

### 4. Migration/current-session handling

The current process may still have an in-memory `_github_token` before restart.

If it is safely possible to migrate that existing in-memory value into the new persistent store WITHOUT printing/logging it, do so.

If it is not safely possible, do NOT fabricate success. Set:
`RECONNECT_ONCE_REQUIRED=YES`

Then the human Super Admin will paste the PAT one final time after FIX 16. The new implementation must persist it from then onward.

### 5. Persistent connection metadata

Persist only safe metadata needed by the UI, for example:

- verified GitHub login
- verified_at timestamp
- token_set boolean derived from secret existence
- selected repository
- selected branch / initial target branch

Do not persist fake sync values.

If selected repository was stale from previous mock data, keep FIX 14 stale-selection invalidation behavior.

### 6. Restart-survival verification

This is the critical acceptance test.

If a valid token is available after applying the code:

1. Confirm `GET /user` succeeds internally.
2. Confirm repository discovery returns the real GitHub response.
3. Record sanitized `full_name` values only; never token.
4. Restart ONLY the NEW TEA service required to reload/verify persistence.
5. Without reconnecting or re-entering the token, verify again:
   - token_set remains true
   - `/user` succeeds
   - repository discovery succeeds
   - same GitHub login is returned
   - real repository list remains available

If no token survives into the new code and human reconnect is required, verify structurally before restart, return `RECONNECT_ONCE_REQUIRED=YES`, and DO NOT claim restart survival yet. The next user reconnect will be the only remaining step.

### 7. Re-run repository diagnosis after persistence

Only after the token survives/reconnects:

Call GitHub read-only:

- `GET /user`
- `GET /user/repos?per_page=100&page=1&sort=updated&affiliation=owner,collaborator,organization_member`

Record only sanitized metadata:

- HTTP status
- login
- raw repository count
- whether a repository named exactly `TEA` appears
- exact `full_name` for `TEA` if present

Then compare:

GitHub raw count -> TEA backend count -> production UI count.

If GitHub authenticates but returns zero repositories AFTER persistence is proven, then and only then classify `TOKEN_NO_REPO_ACCESS` and show the PAT access remediation from FIX 15.

If GitHub returns `TEA`, it must appear in the dropdown even if empty.

### 8. Empty TEA repository behavior remains required

Preserve FIX 14/FIX 15 behavior:

- empty repository is valid
- no fake branches
- initial target branch suggestion `main`
- Review Push supports initial push
- Pull/Sync blocked before first remote history
- actual Initial Push MUST NOT be executed by this patch executor

### 9. Scope protection

Preserve FIX 13 full GitHub module UI.

Do NOT modify unless strictly necessary:

- nginx
- systemd unit definitions
- ports
- React Router/dependencies
- old TEAgent server
- normal OpenHands conversations
- unrelated Developer Hub cards

No new service/daemon/database.

Executor restrictions:

- no git commit/push/pull/fetch/merge/rebase/reset/clean against TEA source
- no Initial Push
- no token output
- GitHub REST read-only diagnostics are allowed

## Validation

Before any restart:

- backend syntax/import validation PASS
- relevant backend tests if present
- frontend type/build only if frontend changed

After restart:

- Developer Hub domain works
- normal OpenHands UI works
- persisted token connection state remains truthful

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_16_PERSIST_GITHUB_TOKEN_ACROSS_RESTART
OLD_TOKEN_STORAGE=
NEW_TOKEN_STORAGE=
SECRET_IN_GIT_TRACKED_FILE=YES|NO
SECRET_FILE_MODE=
TOKEN_RETURNED_TO_BROWSER=YES|NO
TOKEN_LOGGED=YES|NO
CONNECT_PERSISTS_TOKEN=PASS|FAIL
DISCONNECT_DELETES_TOKEN=PASS|FAIL
RECONNECT_ONCE_REQUIRED=YES|NO
TOKEN_SET_BEFORE_RESTART=YES|NO|NOT_AVAILABLE
TOKEN_SET_AFTER_RESTART=YES|NO|NOT_TESTED
TOKEN_RESTART_SURVIVAL=PASS|FAIL|NOT_TESTED_RECONNECT_REQUIRED
GITHUB_USER_HTTP=
GITHUB_LOGIN=
GITHUB_REPOS_HTTP=
GITHUB_RAW_REPO_COUNT=
TEA_BACKEND_REPO_COUNT=
UI_REPO_COUNT=
TEA_RETURNED_BY_GITHUB=YES|NO|NOT_TESTED
TEA_REPO_FULL_NAME=
ROOT_CAUSE_AFTER_PERSISTENCE=TOKEN_PERSISTENCE_FIXED|TOKEN_NO_REPO_ACCESS|TEA_FILTER_DROPPED_REPOS|FRONTEND_DROPPED_REPOS|NOT_TESTED_RECONNECT_REQUIRED|OTHER
EMPTY_REPO_STATE=PASS|FAIL|NOT_REACHED
INITIAL_PUSH_REVIEW=PASS|FAIL|NOT_REACHED
INITIAL_PUSH_EXECUTED=NO
SUPERADMIN_GATE=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL|NOT_CHANGED
BUILD_VALIDATION=PASS|FAIL|NOT_CHANGED
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=
STATUS=READY|RECONNECT_ONCE_REQUIRED|FAILED
```

`STATUS=READY` requires actual token restart survival proof and truthful repository discovery.

If the implementation is complete but the old in-memory token cannot be safely migrated and the human must enter it one final time, return `STATUS=RECONNECT_ONCE_REQUIRED`. Do not mislabel that as a PAT permission problem.