# Verify — TEA Developer Hub GitHub UI TCRM Parity V1

Run after applying `GITHUB_UI_TCRM_PARITY_V1.md`.

## UI

- `/developer-hub` shows interactive GitHub connection controls, not status-only.
- Token field supports show/hide before submit.
- After successful verification the token field is cleared and raw token is never returned.
- Connected account/login and permission state are visible.
- Repository list loads after connection.
- Repository selector works and can be refreshed.
- Branch list is scoped to selected repository and can be refreshed.
- Repository + branch selection can be explicitly saved.
- Disconnect works and clears connection UI state without deleting source.
- Push, Pull and Sync each begin with Review/Preview.
- Review displays ahead/behind, dirty/sync state, expected action, files, conflicts/blockers and Push Control result.
- Execute is unavailable before valid Review and enabled only when Review is safe.
- Progress/current step/sanitized logs display during long operation.

## Review integrity

Create a Review, then change at least one state identifier (selected branch/repository/HEAD or equivalent test fixture). The old Execute authorization must be rejected and require a new Review.

## Authorization

- anonymous API request → 401
- authenticated Super Admin → allowed
- authenticated non-Super-Admin → 403
- direct backend API cannot bypass the UI guard

## Security

- raw GitHub credential does not appear in status, connection, repo/branch, preview, operation, audit or export responses
- arbitrary local path is rejected/not accepted
- invalid branch/repository inputs are rejected
- no force push action exists
- no destructive reset/clean action exists
- no arbitrary shell field exists
- mutating operations are serialized/locked
- repeated Execute cannot run the same reviewed mutation twice

## Persistence/runtime

- custom backend imports resolve from `/opt/TBA/openhands/...`, never `/root/.cache/uv/...`
- restart NEW TEA services and verify GitHub module still works
- production domain works through `https://tea.tamiyouz.com/developer-hub`
- normal OpenHands home and conversation UI still work

If any check fails, return `STATUS=FAILED` and fix only the smallest confirmed Developer Hub GitHub issue before rerunning verification.