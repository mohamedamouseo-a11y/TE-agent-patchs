# Developer Hub API Contract

This is a framework-neutral contract. Map it onto TEAgent's existing API conventions and data types.

All routes require canonical **Super Admin** authorization.

## Read endpoints

### Status

`GET /api/superadmin/developer-hub/status`

Example response:

```json
{
  "enabled": true,
  "repository": "TEAgent",
  "branch": "teagent/self-dev-123",
  "headSha": "abc123...",
  "remote": "origin",
  "defaultBranch": "main",
  "directDefaultBranchPushAllowed": false,
  "worktree": {
    "clean": false,
    "staged": 1,
    "unstaged": 2,
    "untracked": 0
  },
  "ahead": 1,
  "behind": 0
}
```

Do not return repository credentials or sensitive machine configuration.

### Diff

`GET /api/superadmin/developer-hub/diff`

Return changed files, stats, and safe diff text. Secret-sensitive files return metadata with `redacted: true` and no content.

### Branches

`GET /api/superadmin/developer-hub/branches`

Return safe local/remote branch metadata needed by the UI.

### Audit

`GET /api/superadmin/developer-hub/audit`

Return sanitized Developer Hub audit history.

## Mutation envelope

Every mutating endpoint requires the current expected HEAD:

```json
{
  "expectedHeadSha": "abc123..."
}
```

If it does not match the actual current HEAD immediately before mutation, return HTTP `409` (or the project's canonical conflict equivalent) and perform **no mutation**.

## Branch

`POST /api/superadmin/developer-hub/branch`

```json
{
  "expectedHeadSha": "abc123...",
  "name": "teagent/self-dev-feature-x"
}
```

Validate branch name using Git-safe validation and project policy. Do not interpolate into shell strings.

## Checks

`POST /api/superadmin/developer-hub/checks`

```json
{
  "expectedHeadSha": "abc123...",
  "checkIds": ["lint", "test", "build"]
}
```

`checkIds` must resolve to a server-side allowlist discovered/configured from the TEAgent project. Never accept raw commands from the request.

## Sync

`POST /api/superadmin/developer-hub/sync`

```json
{
  "expectedHeadSha": "abc123...",
  "remote": "origin"
}
```

Use the configured remote only, or a strict server-side allowlist. Do not perform destructive conflict resolution automatically.

## Commit

`POST /api/superadmin/developer-hub/commit`

```json
{
  "expectedHeadSha": "abc123...",
  "message": "feat: add developer hub"
}
```

Require passing required checks for the relevant worktree state. Validate message size/content and stage only policy-approved files. Refuse secret-sensitive files.

## Push

`POST /api/superadmin/developer-hub/push`

```json
{
  "expectedHeadSha": "def456...",
  "branch": "teagent/self-dev-feature-x"
}
```

Feature-branch push only by default. Reject direct default-branch push unless explicitly enabled in trusted server configuration. Never force-push.

## Self-development prepare

`POST /api/superadmin/developer-hub/self-development/prepare`

```json
{
  "expectedHeadSha": "abc123...",
  "task": "Improve Developer Hub status UX"
}
```

Behavior:

1. snapshot branch/HEAD/worktree
2. create/use safe feature branch
3. invoke existing TEAgent/OpenHands task flow scoped to configured repo root
4. return session/task identifier
5. do not deploy or restart the live TEAgent

The task string may guide the coding agent but must never be treated as a shell command.

## Rollback prepare

`POST /api/superadmin/developer-hub/rollback/prepare`

```json
{
  "expectedHeadSha": "def456...",
  "targetSha": "abc123..."
}
```

Phase 00 should return a recovery plan/preview and validate the target. Avoid destructive automatic reset behavior.

## Common mutation response

```json
{
  "ok": true,
  "branch": "teagent/self-dev-feature-x",
  "headBefore": "abc123...",
  "headAfter": "def456...",
  "auditId": "..."
}
```

Errors must be sanitized and must never expose credentials, private keys, raw environment values, or sensitive file contents.
