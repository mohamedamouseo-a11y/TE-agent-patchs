# Phase 00 Test Checklist

Use this after applying the Developer Hub implementation to the actual TEAgent source repository.

## Access control

- [ ] Developer Hub appears in Super Admin navigation.
- [ ] Developer Hub does not appear for normal admin/user roles.
- [ ] Direct URL access by non-Super-Admin is denied.
- [ ] Direct API calls by non-Super-Admin are denied.

## Repository status

- [ ] Current branch is correct.
- [ ] HEAD SHA is correct.
- [ ] Dirty/clean status matches `git status`.
- [ ] Staged/unstaged/untracked counts are correct.
- [ ] Remote/ahead/behind failures degrade safely without breaking the page.

## Diff safety

- [ ] Normal source diffs render correctly.
- [ ] `.env`, credentials, token files and private keys never expose contents.
- [ ] Output/log redaction removes credential-like values.
- [ ] Paths outside configured repo root cannot be read.

## Checks

- [ ] Only discovered/configured project checks are selectable.
- [ ] Arbitrary commands cannot be submitted.
- [ ] Failed required check blocks commit/push.
- [ ] Passing required checks enables the next publication step.

## Git mutation guards

- [ ] Mutating request with stale `expectedHeadSha` returns conflict.
- [ ] No mutation occurs after HEAD conflict.
- [ ] Feature branch creation works.
- [ ] Unsafe branch names are rejected.
- [ ] Direct default-branch push is blocked by default.
- [ ] Force push is not available.
- [ ] Destructive reset/clean is not available.
- [ ] Concurrent mutations are serialized/locked.

## Self Development

- [ ] Starting self-development captures original branch + HEAD.
- [ ] Work happens on a feature branch.
- [ ] Existing TEAgent/OpenHands coding flow is reused.
- [ ] Agent workspace is constrained to configured TEAgent repo root.
- [ ] Resulting diff returns to Developer Hub for review.
- [ ] Checks are required before commit/push.
- [ ] Phase 00 never restarts/replaces/deploys the currently running TEAgent automatically.

## Audit / recovery

- [ ] Successful mutation creates audit record.
- [ ] Failed mutation creates audit record.
- [ ] Audit contains actor/action/time/branch/HEAD before+after/outcome.
- [ ] Audit contains no Git token, password, SSH key or secret contents.
- [ ] Last-known-good reference is captured before mutation.
- [ ] Rollback preparation shows the exact target before any recovery action.

## Regression

- [ ] Existing Super Admin pages still work.
- [ ] Existing authentication/authorization still works.
- [ ] Existing agent tasks still work.
- [ ] Existing build/tests show no new unrelated failures.
