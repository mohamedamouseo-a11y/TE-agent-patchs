# FIX 24 — Provider runtime stability + no self-reload + isolated Initial Push execution

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed production state

After FIX 23, the human Super Admin created a fresh `initial_push` review for `mohamedamouseo-a11y/TEA` and pressed **Execute Reviewed Action**.

Immediately after the click, production showed a React runtime failure again:

`useActiveBackendContext must be used inside <ActiveBackendProvider>`

Current stack identifies the hook consumer as `AppContent` in `src/root.tsx`, proving the FIX 22 split (`App` + `AppContent`) is present but runtime context is still not reliably available.

The user also reports a second stability bug:

- when switching away from the OpenHands browser tab/page and returning, OpenHands reloads itself automatically.

Independent GitHub verification after this event still shows:

- `mohamedamouseo-a11y/TEA` size = 0
- remote branch `main` does not exist

Therefore no initial push has succeeded yet.

## Goal

Fix the real runtime cause of BOTH:

1. recurring `ActiveBackendProvider` context failure
2. unintended OpenHands self-reload / full-page reload when returning to the tab

Then harden the human Initial Push execution so Git commit/hook work cannot mutate or destabilize the live `/opt/TBA` source tree that is currently serving the OpenHands UI.

Do NOT execute the Initial Push in this patch task.

---

## Part A — Prove the provider/runtime failure

Inspect all real render entrypoints and provider definitions before editing:

- `/opt/TBA/src/root.tsx`
- `/opt/TBA/src/contexts/active-backend-context.tsx`
- `/opt/TBA/src/entry.client.tsx` (or actual client entry)
- `/opt/TBA/src/entry.server.tsx` / SSR entry if present
- React Router root/layout definitions
- any `AgentServerUIProviders` / provider aggregator

Prove:

- exact component that calls `useActiveBackendContext()`
- exact provider instance that should own it
- whether client and server entrypoints use the same provider tree
- whether there are duplicate `ActiveBackendContext` module instances caused by HMR/module graph duplication
- whether provider placement differs between first load, hard reload, route navigation, and hydration
- whether FIX 22 created a nested/self-wrapper that is not authoritative

Do NOT disable the hook guard and do NOT add fake default context values.

### Required provider behavior

There must be one canonical provider boundary that reliably wraps every consumer on:

- first load
- hard reload
- `/`
- `/developer-hub`
- route navigation
- returning to the tab after it was hidden/backgrounded
- client hydration (if SSR is used)

If client and server have separate render trees, both must be valid.

---

## Part B — Diagnose and stop the self-reload

The user reports OpenHands reloads itself after switching away and returning.

Determine the real reload trigger. Inspect/verify at least:

- Vite HMR client / websocket activity
- whether production is serving a Vite development server (`/src/...`, `/node_modules/.vite/...` stack paths are currently visible)
- `document.visibilitychange` / focus handlers
- service worker reload/update behavior if present
- backend health/reconnect logic
- React Router revalidation/navigation behavior
- service/process restarts
- source-file mtimes during Developer Hub Execute
- whether `git add` / Husky / `lint-staged` modifies live source files and causes HMR invalidation/full reload

Produce proof of the actual trigger; do not guess.

### Required reload behavior

Returning to the tab must NOT cause a full-page reload unless an explicit user action or unavoidable browser recovery requires it.

If the cause is Vite HMR reacting to Developer Hub Git operations, fix the Git execution architecture rather than hiding HMR errors.

Do NOT globally disable useful error handling just to mask the symptom.

---

## Part C — Isolate human Initial Push execution from the live source tree

The current human Execute path stages/commits in the same `/opt/TBA` working tree that serves the live OpenHands UI.

That is unsafe because Husky/lint-staged can touch files, trigger Vite HMR, and destabilize the running UI during a 2200+ file initial commit.

Repair the Execute implementation so a future HUMAN-approved Initial Push uses an isolated temporary Git execution workspace (for example a safe temporary worktree/staging clone or equivalent reviewed-tree snapshot) instead of mutating the live `/opt/TBA` working tree.

### Isolation requirements

For a valid fresh review with `expected_action=initial_push`:

- live `/opt/TBA` source tree must remain unchanged by staging/commit hooks
- commit creation/hook execution must happen in an isolated temporary workspace
- reviewed file/tree integrity must be preserved
- token/remote URL handling must remain secret-safe
- no force push
- no destructive reset/clean against live source
- stale-review protection remains enforced
- commit failure blocks push
- remote verification remains required before success (FIX 20)
- 240s bounded commit timeout/process-group cleanup remains preserved (FIX 23) unless diagnostics justify a safer value
- temporary workspace must be deleted safely after success/failure
- failure cleanup must not touch live source or remote history

Do not bypass Husky hooks blindly. If hooks are intentionally run, run them inside the isolated workspace.

If the existing Git structure makes `git worktree` inappropriate, use another safe isolation mechanism, but prove the live source is not mutated during execution.

---

## Part D — Recovery state before handoff

The previous Execute attempt must NOT be trusted as success.

Before completing FIX 24, verify read-only:

- remote TEA still empty
- remote main still absent
- no successful initial-push commit exists remotely
- persisted repo remains `mohamedamouseo-a11y/TEA`
- persisted target branch remains `main`
- current review is considered stale after code changes and a fresh review is required

Do NOT execute Push/Commit while applying this patch.

---

## Scope / restrictions for patch executor

Allowed:

- inspect files/logs/processes
- edit only files necessary for this fix
- parse/type/build/backend validation
- browser/runtime verification
- read-only Git commands
- read-only GitHub API calls

Forbidden during patch application:

- git commit
- git push
- git pull
- git fetch
- merge/rebase/reset/clean
- Developer Hub Execute endpoint
- Initial Push
- printing GitHub token or secrets

Do NOT modify unrelated Developer Hub cards, nginx, ports, old TEAgent server, or unrelated OpenHands features unless the reload diagnosis proves a directly required minimal runtime config change.

---

## Validation

### Provider/runtime

- parse/type/build PASS
- `/` loads without Application Error
- `/developer-hub` loads without Application Error
- hard reload PASS
- navigate `/` -> `/developer-hub` -> `/` -> `/developer-hub` PASS
- background/return-to-tab test repeated at least 3 times: NO unintended full reload and NO context error
- normal OpenHands sidebar/conversations UI remains functional

### Git execute architecture

Without executing any commit/push:

- prove future initial-push execution path uses isolated workspace
- prove live `/opt/TBA` is not the staging/commit cwd
- prove hooks run only in isolated workspace if hooks are used
- prove temp cleanup exists for success/failure
- prove FIX20 remote verification remains mandatory
- prove FIX23 timeout/process-group cleanup remains present

### GitHub state

- remote still empty
- remote main absent
- fresh Review Push can be created after code fix
- expected action = `initial_push`
- actual Initial Push remains unexecuted

---

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_24_PROVIDER_RUNTIME_STABILITY_AND_ISOLATED_INITIAL_PUSH
PROVIDER_ROOT_CAUSE=
CLIENT_PROVIDER_TREE=PASS|FAIL
SERVER_PROVIDER_TREE=PASS|FAIL|N/A
DUPLICATE_CONTEXT_INSTANCE=YES|NO
HOOK_GUARD_PRESERVED=PASS|FAIL
HOME_UI=PASS|FAIL
DEVELOPER_HUB_UI=PASS|FAIL
HARD_RELOAD=PASS|FAIL
ROUTE_NAVIGATION_REPEAT=PASS|FAIL
TAB_BACKGROUND_RETURN_REPEAT=PASS|FAIL
SELF_RELOAD_ROOT_CAUSE=
UNINTENDED_FULL_RELOAD_GONE=YES|NO
VITE_HMR_INVOLVED=YES|NO
LIVE_SOURCE_MUTATION_TRIGGER_CONFIRMED=YES|NO
INITIAL_PUSH_EXECUTION_ISOLATED=PASS|FAIL
ISOLATED_WORKSPACE_TYPE=
LIVE_TBA_USED_AS_COMMIT_CWD=YES|NO
HOOKS_RUN_IN_ISOLATED_WORKSPACE=YES|NO|N/A
TEMP_WORKSPACE_CLEANUP=PASS|FAIL
FIX20_REMOTE_VERIFICATION_PRESERVED=PASS|FAIL
FIX23_TIMEOUT_CLEANUP_PRESERVED=PASS|FAIL
REMOTE_EMPTY=YES|NO
REMOTE_MAIN_EXISTS=YES|NO
PERSISTED_REPO=
PERSISTED_BRANCH=
FRESH_INITIAL_PUSH_REVIEW=PASS|FAIL
FRESH_REVIEW_EXPECTED_ACTION=
INITIAL_PUSH_EXECUTED_BY_PATCH=NO
SUPERADMIN_GATE=PASS|FAIL
TOKEN_SECRET_SAFE=PASS|FAIL
PARSE_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL
BUILD_VALIDATION=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires the provider error to be gone, tab-return self-reload to be gone, and the future Initial Push commit path to be isolated from the live `/opt/TBA` working tree while the actual push remains unexecuted.
