# FIX 22 — ActiveBackendProvider regression repair

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed production failure

After FIX 21 / backend restart, production now crashes before rendering the Developer Hub.

Exact browser error:

`Error: useActiveBackendContext must be used inside <ActiveBackendProvider>`

Stack proves the failing hook call is inside:

- `/opt/TBA/src/contexts/active-backend-context.tsx` around line 153
- `/opt/TBA/src/root.tsx` inside `App` around line 338

This is a frontend provider-tree regression. Do NOT touch Developer Hub GitHub sync logic unless required to restore the provider tree.

## Goal

Restore the normal OpenHands provider hierarchy so every component calling `useActiveBackendContext()` is rendered beneath the existing `ActiveBackendProvider`.

The fix must be minimal and preserve all FIX 13–21 Developer Hub work, including the real FIX 20 execute path and persisted TEA selection.

## Inspect first

Before editing, inspect the ACTUAL live source:

- `/opt/TBA/src/root.tsx`
- `/opt/TBA/src/contexts/active-backend-context.tsx`
- imports/usages of `ActiveBackendProvider`
- imports/usages of `useActiveBackendContext`

Capture the relevant provider/component structure around the failing `App` function and the root render/export.

Prove which of these is true:

1. `App` itself calls `useActiveBackendContext()` but `App` is the component that creates/wraps `ActiveBackendProvider`, so the hook runs before the provider exists.
2. `ActiveBackendProvider` was accidentally removed from the root tree.
3. `ActiveBackendProvider` is mounted only on some routes and `/developer-hub` bypasses it.
4. duplicate/nested root components caused the hook-consuming component to sit outside the provider.
5. another exact provider-order regression.

Do not guess. Report the exact root cause.

## Required repair

Use the smallest safe structural change.

Preferred pattern when `App` itself currently calls the hook before wrapping the provider:

- split the hook-consuming body into an inner component, e.g. `AppContent` / `AppRoutes`
- keep `ActiveBackendProvider` in the outer root component
- render the inner hook-consuming component inside that provider

Equivalent minimal restructuring is acceptable if it matches the current architecture.

Do NOT:

- bypass or weaken the context hook guard
- change `useActiveBackendContext()` to return fake/default values outside a provider
- add a second unrelated provider implementation
- remove active-backend functionality
- hardcode backend state

## Scope protection

Preserve:

- Developer Hub route
- FIX 20 real initial-push execute path
- FIX 21 persisted repository/branch state
- GitHub token persistence
- Super Admin gate
- normal OpenHands conversations, Customize, Automate and sidebar behavior

Do NOT modify unless proven necessary:

- `/opt/TBA/openhands/automation/developer_hub/router.py`
- `/root/.openhands/tba/github-state.json`
- nginx
- systemd unit definitions
- ports
- dependencies
- old TEAgent server

Executor restrictions:

- no git commit/push/pull/fetch/merge/rebase/reset/clean against TEA source
- no Developer Hub Execute Reviewed Action
- no Initial Push

## Validation gate

Before any restart:

1. frontend parse validation PASS
2. TypeScript/typecheck PASS
3. frontend build PASS

If validation fails, fix only errors introduced by this patch.

After validation, reload/restart only the NEW TEA frontend/service if actually required.

## Production verification

Verify in the real browser:

1. `/` renders normal OpenHands UI without Application Error.
2. `/developer-hub` renders without Application Error.
3. browser console/runtime no longer throws `useActiveBackendContext must be used inside <ActiveBackendProvider>`.
4. sidebar/navigation still renders.
5. Developer Hub still shows persisted repository `mohamedamouseo-a11y/TEA` and target branch `main`.
6. remote remains empty and `main` remains absent remotely.
7. status remains `Remote Empty — Initial Push Required`.
8. Fresh Review Push is still available with `expected_action=initial_push`.
9. DO NOT execute the push.

Capture one real browser screenshot showing `/developer-hub` healthy after the fix.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_22_ACTIVE_BACKEND_PROVIDER_REGRESSION
ROOT_CAUSE=
ROOT_FILE=/opt/TBA/src/root.tsx
CONTEXT_FILE=/opt/TBA/src/contexts/active-backend-context.tsx
PROVIDER_PRESENT_BEFORE=YES|NO
HOOK_CONSUMER_OUTSIDE_PROVIDER_BEFORE=YES|NO
REPAIR_TYPE=
ACTIVE_BACKEND_PROVIDER_TREE=PASS|FAIL
HOOK_GUARD_PRESERVED=PASS|FAIL
PARSE_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL
BUILD_VALIDATION=PASS|FAIL
HOME_UI=PASS|FAIL
DEVELOPER_HUB_UI=PASS|FAIL
APPLICATION_ERROR_GONE=YES|NO
CONTEXT_RUNTIME_ERROR_GONE=YES|NO
SIDEBAR_UI=PASS|FAIL
PERSISTED_REPO=mohamedamouseo-a11y/TEA|OTHER
PERSISTED_BRANCH=main|OTHER
REMOTE_EMPTY=YES|NO
REMOTE_MAIN_EXISTS=YES|NO
SYNC_STATUS_VALUE=
FRESH_INITIAL_PUSH_REVIEW=PASS|FAIL
INITIAL_PUSH_EXPECTED_ACTION=initial_push|OTHER
INITIAL_PUSH_EXECUTED=NO
FIX20_REAL_EXECUTE_PATH_PRESERVED=PASS|FAIL
SCREENSHOT_PATH=
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires the provider/runtime error to be gone in the real browser, both `/` and `/developer-hub` healthy, persisted TEA selection preserved, and no initial push executed.
