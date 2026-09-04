# TE Agent Patches

Patch bundles for the TEAgent installation.

## Phases

- `phase-00-developer-hub/` — Super Admin Developer Hub with safe self-development Git workflow.

## Important

This repository stores application-ready patch instructions/specifications, not the TEAgent source itself. Phase bundles are designed to be applied by OpenHands/TEAgent from inside the real TEAgent source checkout so the implementation follows the application's existing framework, routing, authentication, UI, test, and deployment patterns.

## Phase 00 goal

Add a **Developer Hub** visible only to **Super Admin** that lets TEAgent safely work on its own source and manage the Git workflow:

`Edit → Review Diff → Run Checks → Commit → Push`

Safety defaults:

- feature branches by default
- no force-push
- no direct push to the protected/default branch unless explicitly enabled server-side
- expected-HEAD checks before mutations
- no arbitrary shell terminal in the UI
- credentials stay server-side and are never returned to the browser
- audit log for every mutating action
- rollback metadata captured before changes

See `phase-00-developer-hub/OPENHANDS_PROMPT.md` for the executable implementation prompt.
