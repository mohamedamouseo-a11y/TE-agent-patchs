# Phase 00 — Super Admin Developer Hub

## Objective

Add a first-class **Developer Hub** inside the existing TEAgent **Super Admin** area so TEAgent/OpenHands can safely develop TEAgent itself and push its own code to Git without risking an uncontrolled self-update.

The implementation MUST reuse the existing TEAgent architecture. Do not create a parallel admin app, duplicate auth system, or unrelated framework.

## UX scope

Create a `Developer Hub` entry inside the existing Super Admin navigation. It must be completely hidden from non-Super-Admin users and protected again on the backend.

The first version contains:

1. **Repository Status**
   - repository path/name
   - current branch
   - current HEAD SHA
   - remote
   - clean/dirty state
   - staged/unstaged/untracked summary
   - ahead/behind state when available

2. **Changes / Diff**
   - sanitized file list
   - staged and unstaged diff preview
   - changed-line statistics
   - no secret-file content rendering

3. **Safe Checks**
   - run only project-approved lint/test/typecheck/build commands discovered from the existing project
   - show command name, status, duration and sanitized output
   - no arbitrary shell textbox

4. **Git Actions**
   - create/switch feature branch
   - sync/pull safely
   - commit
   - push feature branch
   - direct default-branch push disabled by default
   - never force-push

5. **Self Development**
   - expose an orchestration action for the existing TEAgent/OpenHands coding workflow
   - TEAgent may modify its own checkout, but publication follows:
     `snapshot → edit → diff → checks → commit → push`
   - never restart/replace the live instance as part of Phase 00
   - deployment remains a separate explicit step

6. **Audit / Recovery**
   - record actor, action, timestamp, branch, HEAD before/after, outcome
   - record a last-known-good reference before mutating Git state
   - provide rollback preparation/status; no destructive automatic rollback without explicit confirmation

## Authorization

Every Developer Hub route and mutating API MUST require the application's canonical Super Admin authorization check. Hiding the menu alone is not authorization.

## Security rules

- Git credentials/tokens/SSH material remain server-side.
- Never return credentials to the browser or logs.
- Never allow the browser to submit arbitrary filesystem paths.
- Repository root comes from trusted server configuration or the existing application checkout.
- Reject path traversal and operations outside the configured repo root.
- Redact known secret/config files from diff content (`.env*`, credentials, private keys, tokens, secret stores) while still allowing a safe filename-level notice when appropriate.
- Mutating requests include `expectedHeadSha`; reject with conflict if HEAD changed.
- No `git reset --hard`, `git clean -fd`, `git push --force`, or equivalent destructive operation in Phase 00.
- Refuse commit/push while required checks fail unless an existing, audited Super Admin override system already exists and is reused.

## Configuration defaults

Suggested server-side configuration names; adapt to the project's existing config naming conventions rather than introducing a conflicting system:

```env
DEVELOPER_HUB_ENABLED=true
DEVELOPER_HUB_REPO_PATH=/path/to/teagent
DEVELOPER_HUB_REMOTE=origin
DEVELOPER_HUB_DEFAULT_BRANCH=main
DEVELOPER_HUB_ALLOW_DIRECT_PUSH=false
```

Do not commit real secrets or machine-specific credentials.

## Deliverables

- Super Admin navigation + page
- backend Git service using safe process execution
- Super Admin-only API routes
- self-development orchestration integration with existing TEAgent/OpenHands capabilities
- audit persistence using the project's existing database/storage pattern
- tests for authorization, HEAD conflicts, protected branch behavior, redaction, checks, commit/push guards
- no unrelated refactors

Apply using `OPENHANDS_PROMPT.md` and validate using `TEST_CHECKLIST.md`.
