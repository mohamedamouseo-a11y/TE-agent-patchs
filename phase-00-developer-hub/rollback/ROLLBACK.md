# Phase 00 Rollback / Recovery

The Developer Hub is intentionally designed so self-development does not automatically replace the live TEAgent runtime.

## Before applying the phase

Record:

```bash
git branch --show-current
git rev-parse HEAD
git status --short
```

Keep the original branch and HEAD as the recovery reference.

## If the implementation is still uncommitted

Use normal Git review/revert workflows appropriate to the TEAgent repository. Do not use destructive reset/clean commands through the Developer Hub.

## If the implementation was committed on a feature branch

Leave the default branch unchanged and stop using the feature branch. If a revert is required, create a normal revert commit rather than rewriting history.

## If the feature branch was pushed

The pushed branch can remain for investigation/review. Do not force-push or rewrite the protected/default branch.

## If the change was later merged/deployed outside Phase 00

Use the application's established deployment rollback procedure. Prefer a revert commit to a known-good revision and redeploy through the normal deployment mechanism.

## Recovery invariants

- never expose or copy Git credentials during recovery
- never delete the only known-good revision
- never let a self-development task automatically restart the live controlling instance
- preserve audit records for failed and recovered operations
- confirm the exact target SHA before any rollback action
