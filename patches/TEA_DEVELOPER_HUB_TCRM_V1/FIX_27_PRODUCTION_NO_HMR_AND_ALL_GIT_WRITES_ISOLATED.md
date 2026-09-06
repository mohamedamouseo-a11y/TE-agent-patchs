# FIX 27 — Production no-HMR runtime + isolate ALL Developer Hub Git writes

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed production problem

FIX 24 solved isolation only for the `initial_push` path, but the user still sees OpenHands reload itself when switching to another page/tab and returning.

Current production evidence also shows a later ordinary `PUSH` operation running Husky/lint-staged behavior (`Backing up original state...`, staged-file tasks) and failing while creating `Sync changes`. This proves the normal post-initial-push write path is still capable of operating against the live `/opt/TBA` checkout.

Independent source evidence from TEA `package.json` shows:
- development frontend: `react-router dev` (Vite/HMR enabled)
- production build: `npm run build`
- production static start: `npx sirv-cli build/ --single`

A production domain must not depend on Vite HMR or a live-source watcher.

## Goal

Close the reload problem architecturally, not symptom-by-symptom:

1. production `tea.tamiyouz.com` frontend must run from a built production artifact with NO Vite HMR client/watch loop;
2. ALL Developer Hub Git write actions must execute in isolated temporary workspaces, never in the live `/opt/TBA` checkout;
3. preserve automation/API routing, GitHub connection, Developer Hub functionality, and all previous safety gates.

Do not execute any new Push/Pull/Sync while applying this patch.

---

## Part A — Prove the current runtime before editing

Inspect the actual production runtime, not assumptions:

- active systemd/service unit(s) serving TEA
- process tree and exact command line
- current listening ports
- nginx/ingress upstream for `tea.tamiyouz.com`
- whether the frontend process is `react-router dev`, Vite dev server, `scripts/dev-with-automation.mjs`, `sirv`, `react-router-serve`, or something else
- whether `/api/automation` is served by the same process or a separate backend process

Verify from the browser/network/source whether production currently loads any of:
- `/@vite/client`
- Vite HMR websocket
- React Refresh/HMR runtime

Record exact root cause.

## Part B — Production frontend must not use Vite dev/HMR

Repair the runtime so the public production frontend is served from a production build.

Requirements:

1. Run the project-supported production build (`npm run build`) and validate it before switching runtime.
2. Use the project's supported production serving mechanism (the repository currently defines `npm run start` as `sirv-cli build/ --single`) OR an equivalent existing production React Router serve path only if the current deployment architecture requires SSR.
3. Preserve the public URL and existing nginx/ingress contract.
4. Preserve `/api/automation` routing to the automation backend.
5. Do not remove/disable the automation backend merely to make the frontend static.
6. If the current wrapper script starts both frontend and automation, split or adjust the service architecture safely so:
   - frontend is production-built/non-HMR;
   - automation backend remains available on its existing internal route/port;
   - public `tea.tamiyouz.com` behavior remains unchanged except reload/HMR removal.
7. Do not expose a new public port.
8. Do not leave the production domain running `react-router dev`, `vite`, or any HMR-enabled dev command.
9. Use atomic deployment / safe service restart. Do not leave the site down if build/service validation fails.

After the switch, production HTML/network must not contain `/@vite/client` and must not open a Vite HMR websocket.

---

## Part C — Isolate ALL Developer Hub write paths

FIX 24 isolation must become a single reusable write-execution mechanism for every Git-mutating Developer Hub action.

Inspect all action branches in `/opt/TBA/openhands/automation/developer_hub/router.py` and related helpers.

At minimum cover:
- initial_push
- normal push
- sync action when it creates a local commit and/or pushes
- any cleanup/recovery path that stages/commits files

### Rule

No Git-mutating command may use `/opt/TBA` as its cwd.

Forbidden in live `/opt/TBA` during Execute:
- `git add`
- `git commit`
- `git stash`
- `git checkout` / switch that mutates files
- `git reset`
- `git clean`
- hook execution / lint-staged
- push preparation that rewrites/indexes the live working tree

Read-only Git status/diff/log/HEAD inspection is allowed in `/opt/TBA`.

### Reusable isolated execution

Refactor/reuse the FIX 24 temporary worktree strategy so ordinary push uses it too:

1. create a detached temp worktree from the exact reviewed base HEAD;
2. materialize the exact reviewed source snapshot/change set into the isolated worktree;
3. verify repo/branch/review fingerprint binding before staging;
4. stage only the reviewed intended tree/files according to existing push policy;
5. run Husky/lint-staged and commit inside the isolated workspace only;
6. preserve commit timeout/process-group cleanup from FIX 23;
7. push with validated fully-qualified destination refspec from FIX 25;
8. verify remote branch/commit/tree using FIX 20/FIX 26 authoritative verification;
9. clean up the temporary worktree in success and failure paths;
10. never mutate the live `/opt/TBA` index or working files.

If a push requires local changes that exist only in the live working tree, snapshot/copy the reviewed versions into the isolated workspace. Do NOT stage them in the live repository as an intermediate step.

### Pull / sync caveat

If a Pull/Sync action is intended to update the live deployed source, do not silently mutate a dev-watched source tree. With Part B, the public UI must already run from a production build, so source checkout changes must not trigger browser HMR.

Still preserve explicit human Review -> Execute gates and existing conflict/stale protections.

---

## Part D — Fix current stale/partial Git state safely

The previous failed normal PUSH may have left live index state or hook artifacts.

Inspect read-only first:
- staged count
- unstaged count
- untracked count
- live HEAD
- remote HEAD
- running git/husky/lint-staged processes

Do not run destructive reset/clean.

If stale index state was created only by Developer Hub's failed operation, normalize it only using a non-destructive method that preserves every working-tree byte. Report exactly what was done. If safe ownership cannot be proven, leave it untouched and report a blocker.

No remote write is allowed during this FIX task.

---

## Part E — Verification: reload must be genuinely gone

Do not mark READY from source inspection alone.

### Runtime checks

Verify:
- `/` loads normally
- `/developer-hub` loads normally
- normal OpenHands routes load normally
- `/api/automation` remains functional
- production frontend has no Vite HMR client
- production frontend has no Vite HMR websocket

### Browser stability test

Using an actual browser session if available:

1. open `/developer-hub`;
2. record a unique in-memory marker and current navigation/performance state;
3. navigate inside OpenHands to another route and back at least 5 times;
4. background the browser tab, wait, foreground it, repeat at least 5 times;
5. confirm there is no full document reload and in-memory marker remains;
6. wait at least 30 seconds idle and repeat focus return;
7. confirm no recurring `useActiveBackendContext` error.

If no browser session exists, prove all server/runtime/HMR checks and return `BROWSER_STABILITY=NOT_TESTED_NO_BROWSER` instead of claiming PASS.

### Developer Hub safety verification

Create Review only if needed, but DO NOT Execute.

Verify source/code paths prove:
- initial_push uses isolated workspace
- normal push uses isolated workspace
- sync commit/push uses isolated workspace
- live `/opt/TBA` is never commit cwd
- remote verification is preserved
- no force push
- token is never logged

---

## Strict restrictions while applying FIX 27

Do NOT:
- call Execute Reviewed Action
- `git push`
- `git pull`
- `git fetch`
- create a Git commit for TEA source
- force push
- reset/clean the live tree
- print tokens/secrets

Service/build/runtime changes required to move production off Vite dev are allowed after validation.

---

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_27_PRODUCTION_NO_HMR_AND_ALL_GIT_WRITES_ISOLATED
CURRENT_FRONTEND_RUNTIME_BEFORE=
CURRENT_SERVICE_UNIT=
CURRENT_PUBLIC_UPSTREAM=
VITE_HMR_PRESENT_BEFORE=YES|NO
SELF_RELOAD_ROOT_CAUSE=
PRODUCTION_BUILD=PASS|FAIL
PRODUCTION_FRONTEND_RUNTIME_AFTER=
VITE_HMR_PRESENT_AFTER=YES|NO
VITE_HMR_WEBSOCKET_AFTER=YES|NO
API_AUTOMATION_PRESERVED=PASS|FAIL
HOME_UI=PASS|FAIL
DEVELOPER_HUB_UI=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
ROUTE_NAVIGATION_5X=PASS|FAIL|NOT_TESTED_NO_BROWSER
TAB_BACKGROUND_RETURN_5X=PASS|FAIL|NOT_TESTED_NO_BROWSER
IDLE_FOCUS_RETURN=PASS|FAIL|NOT_TESTED_NO_BROWSER
FULL_DOCUMENT_RELOAD_OBSERVED=YES|NO|NOT_TESTED_NO_BROWSER
ACTIVE_BACKEND_CONTEXT_ERROR=YES|NO|NOT_TESTED_NO_BROWSER
INITIAL_PUSH_ISOLATED=PASS|FAIL
NORMAL_PUSH_ISOLATED=PASS|FAIL
SYNC_WRITE_PATH_ISOLATED=PASS|FAIL
LIVE_TBA_USED_AS_GIT_WRITE_CWD=YES|NO
HOOKS_RUN_IN_LIVE_TBA=YES|NO
TEMP_WORKSPACE_CLEANUP=PASS|FAIL
FIX23_TIMEOUT_PRESERVED=PASS|FAIL
FIX25_FULL_REFSPEC_PRESERVED=PASS|FAIL
FIX26_REMOTE_VERIFICATION_PRESERVED=PASS|FAIL
LIVE_STAGED_COUNT_BEFORE=
LIVE_STAGED_COUNT_AFTER=
LIVE_WORKTREE_BYTES_PRESERVED=YES|NO
REMOTE_WRITE_EXECUTED_BY_PATCH=NO
TOKEN_SECRET_SAFE=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL|NOT_CHANGED
BUILD_VALIDATION=PASS|FAIL
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires production HMR to be removed and ALL Git write paths to be isolated. Do not return READY if only one side is fixed.
