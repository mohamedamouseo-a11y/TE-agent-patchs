# OpenHands Prompt — TEAgent Phase 00 Developer Hub

Implement **Phase 00: Super Admin Developer Hub** in the TEAgent repository you are currently running from.

## Non-negotiable first step

Before changing anything, inspect the repository and identify:

- current Super Admin layout/navigation/routes
- canonical Super Admin authorization guard on frontend and backend
- backend router/service patterns
- config/env pattern
- existing Git/process-execution helpers, if any
- existing OpenHands/agent task execution flow
- test commands from package/project configuration
- existing audit/log persistence pattern

Use the existing architecture. Do not introduce a second framework or duplicate authentication system.

## Feature

Add a **Developer Hub** inside the existing **Super Admin** area and make it inaccessible to every non-Super-Admin role.

The page should provide these sections:

1. **Repository Status** — branch, HEAD SHA, remote, clean/dirty state, staged/unstaged/untracked counts, ahead/behind when available.
2. **Changes** — safe file list and staged/unstaged diff preview with secret-file contents redacted.
3. **Checks** — run the repository's approved lint/test/typecheck/build commands only. Do not expose arbitrary shell execution.
4. **Git Actions** — create/switch feature branch, safe sync, commit, push feature branch.
5. **Self Development** — connect to the existing TEAgent/OpenHands coding flow so TEAgent can modify its own checkout. Publication must follow `snapshot → edit → diff → checks → commit → push`.
6. **Audit / Recovery** — show recent Developer Hub actions and capture HEAD/branch before and after every mutation.

## Backend contract

Adapt endpoint names to the project's existing API conventions, but implement equivalent capabilities:

- `GET /api/superadmin/developer-hub/status`
- `GET /api/superadmin/developer-hub/diff`
- `GET /api/superadmin/developer-hub/branches`
- `GET /api/superadmin/developer-hub/audit`
- `POST /api/superadmin/developer-hub/branch`
- `POST /api/superadmin/developer-hub/checks`
- `POST /api/superadmin/developer-hub/sync`
- `POST /api/superadmin/developer-hub/commit`
- `POST /api/superadmin/developer-hub/push`
- `POST /api/superadmin/developer-hub/self-development/prepare`
- `POST /api/superadmin/developer-hub/rollback/prepare`

Every mutating request MUST include `expectedHeadSha`. Compare it to the current HEAD immediately before the operation and return a conflict if it changed.

## Git safety policy

Implement these rules server-side, not only in UI:

- repo path is trusted server configuration / current TEAgent checkout; never accept arbitrary repo path from browser
- default remote is configurable, normally `origin`
- default branch is configurable, normally `main`
- direct push to default branch is disabled by default
- feature-branch push is the standard path
- no force push
- no destructive reset/clean
- no automatic live deployment/restart in this phase
- do not commit or display secrets
- redact `.env*`, credentials, tokens, private keys and project secret stores from diff bodies/logs
- sanitize process output before persistence/response
- use argument-array process execution or an existing safe wrapper; do not construct shell commands from user input
- validate branch names and commit messages
- serialize or lock mutating Git actions so two requests cannot race

Before a mutating Git action, capture at least:

- actor user id
- action
- branch
- HEAD SHA before
- timestamp

After it, capture:

- HEAD SHA after
- success/failure
- sanitized error/result

## Checks gate

Discover the project's real commands from its package/build configuration. Register only known commands such as lint, tests, typecheck, and build that actually exist in this repository.

The UI may select from this allowlist. It must never accept a raw shell command.

Commit/push must be blocked when required checks for the current HEAD/worktree have failed or have not run, unless the existing application already provides an audited Super Admin override mechanism; if so, reuse it instead of inventing a bypass.

## Self-development behavior

Integrate with the existing TEAgent/OpenHands agent/task mechanism rather than building a second coding agent.

When Super Admin starts a self-development session:

1. capture current branch + HEAD + worktree status
2. require or create a feature branch such as `teagent/self-dev-<timestamp-or-id>`
3. hand the requested development task to the existing coding-agent flow scoped to the TEAgent repo root
4. never allow the task to escape the configured repo root
5. when agent editing completes, return to Developer Hub with the resulting diff
6. require checks before commit/push
7. push the feature branch only after explicit Super Admin action
8. do NOT restart, replace, pull into, or deploy over the currently running TEAgent instance in Phase 00

This separation is critical: **TEAgent may write its own source, but it must not automatically replace the live runtime that is currently controlling the operation.**

## UI requirements

Reuse the current Super Admin design system, components, spacing, typography and dark/light mode behavior.

Add a clear status header with branch + shortened SHA + worktree state. Dangerous or publication actions should require explicit confirmation and show exactly what branch/HEAD will be affected.

Do not show Developer Hub menu/routes to unauthorized users. A manually entered URL must still receive the application's normal forbidden/unauthorized behavior.

## Tests

Add tests using the repository's existing test framework. At minimum cover:

- Super Admin can access; normal users cannot
- backend rejects non-Super-Admin even if endpoint called directly
- expected HEAD mismatch returns conflict and performs no mutation
- default branch direct push is blocked by default
- force-push/destructive commands are unavailable
- branch validation rejects unsafe names/input
- repo-root escape is impossible
- secret-file diff contents are redacted
- arbitrary shell command cannot be submitted
- checks gate blocks commit/push when checks fail
- audit entry records success and failure without credentials
- self-development prepares a feature branch and does not deploy/restart live TEAgent

## Completion protocol

After implementation:

1. run the relevant project tests/checks
2. show `git status --short`
3. show current branch and HEAD
4. list every changed file
5. summarize tests/checks with PASS/FAIL
6. do not fix unrelated pre-existing failures
7. do not push to the default branch
8. if Git credentials are already configured and Super Admin explicitly requested publication, push the feature branch only

Return a compact report in this exact shape:

```text
PHASE=00-DEVELOPER-HUB
HEAD_BEFORE=<sha>
HEAD_AFTER=<sha>
BRANCH=<branch>
SUPERADMIN_GUARD=PASS|FAIL
HEAD_CONFLICT_GUARD=PASS|FAIL
DEFAULT_BRANCH_PROTECTION=PASS|FAIL
SECRET_REDACTION=PASS|FAIL
CHECKS_GATE=PASS|FAIL
SELF_DEVELOPMENT_SAFE_MODE=PASS|FAIL
TESTS=<summary>
CHANGED_FILES=<comma-separated list>
PUSH=<NOT_ATTEMPTED|PUSHED branch sha|FAILED reason>
```
