# TCRM Developer Hub reference contract

Reference repository: `mohamedamouseo-a11y/TCRM-MAIN-Tamiyouz-CRM-`

Pinned commit: `78273711727e834ca88029e39ff0f6ae302d427a`

## Reference source files

Primary:

- `client/src/components/DeveloperHubTab.tsx`
- `server/routes/developerHub.ts`

Related panels/services should be inspected at the same pinned commit when referenced by the primary files, including Git, Push Control, MCP, AI Context, operation recovery, preflight and security helpers.

## UI/behavior to preserve

The TEA port must preserve the TCRM Developer Hub operating model rather than replacing it with a generic Git page.

### Status / connection

Show repository path, current branch, short HEAD, dirty state, unpushed count, GitHub connection status, configured repository, selected branch/default branch, last synchronization information, account/permission state and operational health.

### GitHub connection

- Private repository connection is configured by the human Super Admin.
- Repository and branch selection are manual Super Admin actions.
- Credentials stay on the server.
- Never expose an access token after submission.
- Disconnect must not delete application source or history.

### Review before execute

Push, Pull and Sync follow the TCRM review model:

1. create a preview/review
2. report local ahead / remote ahead / dirty state
3. report changed files and conflicts/blockers
4. show expected action
5. run required preflight/security checks
6. only then allow an explicit human Execute action

A review result must be tied to the state it reviewed (branch/HEAD/fingerprint or equivalent) so stale reviews cannot authorize a different mutation.

### Push Control / preflight

Preserve the equivalent TCRM states for verification, tests, build, security and remote synchronization. A failed mandatory gate blocks publication. Do not invent an invisible bypass.

### Operation progress/recovery

Long Git operations expose stable operation IDs, progress, current step, sanitized logs and final result. Reloading the page must allow a Super Admin to recover the active operation where practical instead of accidentally starting a duplicate mutation.

### Additional Developer Hub areas

Port the current TCRM equivalents for:

- Git controls
- Push Control
- MCP
- AI Context generation/sync controls
- Source Code export
- operation/audit history
- recovery/status indicators

If a TCRM capability depends on a CRM-only subsystem that has no TEA equivalent, keep the Developer Hub section/intent and bind it to TEA's existing equivalent. Do not add an unrelated framework just to duplicate a TCRM implementation detail.

## Backend safety contract

Every backend Developer Hub route is Super Admin-only, enforced server-side.

Required rules:

- trusted fixed TEA repository root; browser cannot select an arbitrary filesystem path
- arguments passed to Git/process execution without shell interpolation from user input
- validate repository URL, branch names and commit messages
- no force push
- no destructive reset/clean workflow
- no secret files in diffs/exports/logs
- serialize/lock mutating operations
- no duplicate operation execution from refresh/retry
- audit successful and failed mutations
- human-triggered Git writes only
- no autonomous agent commit/push/pull/sync
- no Developer Hub restart/deploy controls

## Expected capabilities

Use TEA/OpenHands API conventions, but provide equivalent capabilities for:

- Developer Hub status
- GitHub connection/configuration status
- repository discovery/selection where supported
- branch discovery/selection
- sync preview
- sync execute
- operation status/recovery
- preflight/security status
- source export
- Git manual controls
- Push Control settings
- MCP settings/status
- AI Context settings/actions
- audit/history

## Visual parity

Match the current TCRM Developer Hub information architecture, labels, cards, status badges, progress states, review/execute flow and operational emphasis as closely as practical. Use TEA's existing component/design primitives so the new page remains native to OpenHands and does not import TCRM's UI framework wholesale.