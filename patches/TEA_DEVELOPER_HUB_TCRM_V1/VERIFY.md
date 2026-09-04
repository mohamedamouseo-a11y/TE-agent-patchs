# Verification — TEA Developer Hub TCRM V1

All applicable checks must pass before reporting READY_FOR_REVIEW.

## Access

- Developer Hub appears only for Super Admin in the TEA UI.
- A non-Super-Admin cannot open the route through a manually entered URL.
- Direct calls to every Developer Hub backend route by a non-Super-Admin return the application's normal 401/403 behavior.
- Super Admin authorization is enforced by the backend, not a frontend feature flag alone.

## TCRM parity

- Status area shows repo/branch/HEAD/dirty/sync state.
- GitHub connection state and manual repository/branch selection exist.
- Push / Pull / Sync use Review/Preview before Execute.
- Preview shows relevant ahead/behind, changed-file, conflict/blocker and expected-action information.
- Push Control/preflight status is visible and gates publication.
- Git controls are present.
- MCP controls/status are present.
- AI Context controls/status are present.
- Source Code export is present and protected.
- Operation progress/recovery is present for long operations.
- Audit/history is present for successful and failed write actions.

## Git and security

- Repository root is the configured TEA repository; arbitrary browser-provided filesystem paths are rejected.
- No token/credential/private key is returned to the browser, diff, logs, audit or source export.
- No arbitrary shell command field exists.
- Branch/repository/commit inputs are validated.
- No force push exists.
- No destructive reset/clean action exists.
- A stale Preview cannot authorize an Execute after branch/HEAD/repository state changes.
- Concurrent/retried write operations cannot execute twice.
- Agent tasks cannot autonomously invoke commit/push/pull/sync.
- Every Git write requires a human Super Admin action.

## Runtime safety

- Developer Hub contains no restart/deploy action and cannot restart the TEA instance.
- Existing OpenHands home UI still loads.
- Existing conversation list opens.
- A normal conversation can be opened/created using the pre-existing TEA behavior.
- Existing WebSocket/conversation behavior is not intentionally changed by this patch.
- No OpenHands dependency/version, port, nginx or systemd change was introduced by this patch.

## Final smoke test

Verify `https://tea.tamiyouz.com` after loading the implementation. Any restart required to load the new code must be performed by the external executor against the NEW TEA service only, never from Developer Hub itself.