# FIX 28 — Restore production automation API + complete ALL Git write isolation

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Starting point from FIX 27

FIX 27 produced a PARTIAL result:

- production frontend successfully moved off `react-router dev` / Vite HMR;
- production frontend now runs from built static artifacts via `sirv-cli`;
- Vite HMR client/websocket are gone;
- however `/api/automation` is broken after the runtime switch;
- ordinary `push` and `sync` write paths are still not isolated;
- live `/opt/TBA` is still used as Git write cwd for those paths;
- hooks/lint-staged can still run against live `/opt/TBA`;
- 37 staged files remain in the live checkout from a previous failed operation;
- no remote write was executed by FIX 27.

FIX 27 completion report explicitly returned:

- `API_AUTOMATION_PRESERVED=FAIL`
- `NORMAL_PUSH_ISOLATED=FAIL`
- `SYNC_WRITE_PATH_ISOLATED=FAIL`
- `LIVE_TBA_USED_AS_GIT_WRITE_CWD=YES`
- `HOOKS_RUN_IN_LIVE_TBA=YES`
- `BACKEND_VALIDATION=FAIL`
- `STATUS=FAILED`

Therefore FIX 28 must complete the architectural migration rather than partially reverting it.

---

## Goal

Achieve all of the following together:

1. keep production frontend on NON-HMR production serving;
2. restore `/api/automation` and all Developer Hub API functionality;
3. isolate every Developer Hub Git-mutating action from `/opt/TBA`;
4. preserve all previous safety/verification behavior;
5. do not execute any remote Git write while applying this patch;
6. preserve all live working-tree bytes exactly.

`STATUS=READY` is forbidden unless BOTH API restoration and complete Git-write isolation pass.

---

## Part A — Diagnose the current production topology first

Inspect the live runtime before changing anything:

- `/etc/systemd/system/tba-openhands.service`
- `/opt/TBA/scripts/prod-with-automation.mjs`
- any automation backend unit/process
- all listening ports related to TEA
- nginx config serving `tea.tamiyouz.com`
- exact nginx `proxy_pass` rules
- current `sirv-cli` process command/port
- current automation API process command/port
- current response for `/api/automation/...` through the public domain
- direct local response from the automation backend internal port if one exists

Prove why `API_AUTOMATION_PRESERVED=FAIL`.

Likely possibilities include, but are not limited to:

- `sirv-cli` is receiving `/api/automation` because nginx proxies the whole domain to one static frontend port;
- the old wrapper previously multiplexed static/frontend + automation on one port and the new production wrapper does not;
- automation backend is not started at all;
- automation backend is listening on a different internal port/path;
- nginx route ordering is incorrect.

Do not guess. Record exact ports and process topology.

---

## Part B — Restore `/api/automation` without reintroducing HMR

Production must remain HMR-free.

### Required architecture

Use a stable split or multiplex architecture that preserves the public contract:

- frontend: production build only, no Vite/react-router dev;
- automation backend: separate long-running backend process or existing internal service;
- nginx (preferred) or an existing production-safe proxy routes:
  - `/api/automation/...` -> automation backend internal listener;
  - all frontend routes -> production frontend listener/static build.

Do NOT route `/api/automation` into `sirv-cli`.

Do NOT restore the old Vite dev wrapper as a shortcut.

### Verification

After the change:

- `GET /` returns frontend normally;
- `GET /developer-hub` returns frontend normally;
- `/api/automation` endpoints used by Developer Hub return the same JSON/API behavior as before;
- GitHub connection state can be read through the public domain;
- repo status can be read through the public domain;
- preview/review API can be read through the public domain;
- no `/@vite/client` appears in HTML/source/network;
- no Vite HMR websocket exists.

If service splitting is necessary, use systemd units with restart policy and explicit dependencies. Keep internal ports bound to localhost where possible.

Run `nginx -t` before reload if nginx changes.

---

## Part C — Create ONE reusable isolated Git-write executor

Inspect `/opt/TBA/openhands/automation/developer_hub/router.py` fully and identify every path that mutates Git state.

Do not only patch the obvious branch.

At minimum cover:

- `initial_push`;
- ordinary `push`;
- `sync` when it stages/commits/pushes;
- any execute helper that runs `git add`, `git commit`, hooks, stash, checkout/switch, reset, clean, or push preparation;
- cleanup/recovery flows that touch the Git index or worktree.

### Hard rule

For Developer Hub execute paths, `/opt/TBA` may be used only for READ-ONLY Git operations such as:

- status;
- diff;
- log;
- rev-parse;
- branch/ref inspection;
- fingerprint calculation.

The following are forbidden with cwd inside `/opt/TBA`:

- `git add`;
- `git commit`;
- `git stash`;
- mutating checkout/switch;
- `git reset`;
- `git clean`;
- Husky hook execution;
- lint-staged execution;
- any commit preparation that writes the live index.

### Required isolated executor behavior

Refactor to a single reusable helper/context-manager used by ALL write paths:

1. create a temp isolated Git worktree/workspace from the exact reviewed base commit;
2. bind it to the selected repo, selected branch, reviewed HEAD and review fingerprint;
3. materialize the reviewed source snapshot/change set into the temp workspace;
4. verify stale protection again before staging;
5. stage intended reviewed files only;
6. run hooks/lint-staged inside temp workspace only;
7. create the commit in temp workspace only;
8. preserve FIX 23 timeout/noninteractive/process-group cleanup;
9. preserve FIX 25 fully-qualified refspec behavior;
10. preserve FIX 20/FIX 26 remote verification behavior;
11. cleanup temp workspace on success, failure, exception and timeout;
12. never leave a Git index/worktree mutation behind in `/opt/TBA`.

For ordinary pushes where working-tree changes exist only in `/opt/TBA`, copy/materialize the reviewed file contents into the temp workspace without staging them in `/opt/TBA` first.

The reviewed file set and fingerprint must remain authoritative.

---

## Part D — Handle current live staged state safely

Current reported state:

- `LIVE_STAGED_COUNT_BEFORE=37`
- working-tree bytes must be preserved exactly.

Inspect the staged paths and determine whether they are solely artifacts of prior Developer Hub failures.

### Safe handling rule

- Do not run `git reset --hard`.
- Do not run `git clean`.
- Do not discard any file content.
- Do not overwrite working-tree bytes.

If and only if every staged entry is proven to be an index-only artifact from the failed Developer Hub flow, clear ONLY the index staging state using a non-destructive operation that preserves working-tree file contents byte-for-byte, for example an index-only unstage operation appropriate to the exact Git state.

Before and after any such normalization:

- hash/compare affected working-tree files;
- confirm bytes unchanged;
- report staged count before/after.

If ownership cannot be proven safely, leave the 37 staged entries untouched and report `LIVE_INDEX_NORMALIZATION=BLOCKED_UNPROVEN_OWNERSHIP` while still completing API/runtime and future-write isolation.

Do not let the presence of old staged state cause the new isolated executor to use the live index.

---

## Part E — Preserve all previous fixes

Must preserve:

- FIX 20 real remote write path + remote success verification;
- FIX 23 bounded commit timeout, noninteractive env, process-group cleanup and hook safety;
- FIX 24 temp-worktree strategy and cleanup;
- FIX 25 fully-qualified push refspec `HEAD:refs/heads/<validated_branch>` where applicable;
- FIX 26 authoritative remote branch/commit/tree verification without relying on repo `size` metadata;
- persisted GitHub token secrecy;
- persisted repo/branch selection;
- Super Admin gate;
- Review -> Execute human gate;
- stale review protection;
- no force push.

---

## Part F — Validation

### Backend/runtime

Run syntax/import/backend validations after code changes.

Validate service topology:

- production frontend process is non-HMR;
- automation backend process is alive;
- public `/api/automation` reaches automation backend;
- nginx config test passes if changed;
- service restart/reload succeeds;
- no public port exposure beyond existing contract.

### Frontend

Use existing production build from FIX 27 unless source changes require rebuild.

If frontend source changes, run the required type/build validation before deployment.

Verify no Vite HMR artifacts are present after service restart.

### Developer Hub read-only verification

Without executing any Git write:

- load repo status;
- load GitHub connection status;
- create a fresh Review Push if needed, but DO NOT Execute;
- verify the review can be created through the restored public API;
- verify code path inspection proves ordinary push and sync will use temp isolated workspace.

### Browser stability

If browser session exists:

- `/developer-hub` loads;
- normal OpenHands UI loads;
- navigate routes and return 5 times;
- background/foreground tab 5 times;
- idle 30 seconds then focus;
- no full document reload;
- no ActiveBackendContext runtime error.

If browser session is unavailable, return NOT_TESTED rather than inventing PASS.

---

## Strict restrictions while applying FIX 28

Do NOT:

- call Execute Reviewed Action;
- `git push`;
- `git pull`;
- `git fetch`;
- create a TEA source commit;
- force push;
- merge/rebase;
- reset/clean live working tree;
- print token/secret values;
- revert production frontend back to Vite/react-router dev.

Service, nginx and backend code changes required to complete FIX 28 are allowed after validation.

---

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_28_PRODUCTION_API_RESTORE_AND_COMPLETE_GIT_ISOLATION
FIX27_FRONTEND_NON_HMR_PRESERVED=PASS|FAIL
API_FAILURE_ROOT_CAUSE=
FRONTEND_RUNTIME=
FRONTEND_INTERNAL_PORT=
AUTOMATION_RUNTIME=
AUTOMATION_INTERNAL_PORT=
NGINX_FRONTEND_UPSTREAM=
NGINX_AUTOMATION_UPSTREAM=
API_AUTOMATION_PUBLIC=PASS|FAIL
GITHUB_CONNECTION_API=PASS|FAIL
REPO_STATUS_API=PASS|FAIL
REVIEW_API=PASS|FAIL
VITE_HMR_PRESENT=YES|NO
VITE_HMR_WEBSOCKET=YES|NO
INITIAL_PUSH_ISOLATED=PASS|FAIL
NORMAL_PUSH_ISOLATED=PASS|FAIL
SYNC_WRITE_PATH_ISOLATED=PASS|FAIL
ALL_GIT_WRITE_PATHS_SHARE_ISOLATED_EXECUTOR=YES|NO
LIVE_TBA_USED_AS_GIT_WRITE_CWD=YES|NO
HOOKS_RUN_IN_LIVE_TBA=YES|NO
TEMP_WORKSPACE_CLEANUP=PASS|FAIL
LIVE_STAGED_COUNT_BEFORE=
LIVE_STAGED_COUNT_AFTER=
LIVE_INDEX_NORMALIZATION=PASS|BLOCKED_UNPROVEN_OWNERSHIP|NOT_NEEDED|FAIL
LIVE_WORKTREE_BYTES_PRESERVED=YES|NO
FIX23_TIMEOUT_PRESERVED=PASS|FAIL
FIX25_FULL_REFSPEC_PRESERVED=PASS|FAIL
FIX26_REMOTE_VERIFICATION_PRESERVED=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
TOKEN_SECRET_SAFE=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL|NOT_CHANGED
BUILD_VALIDATION=PASS|FAIL|NOT_CHANGED
HOME_UI=PASS|FAIL
DEVELOPER_HUB_UI=PASS|FAIL|NOT_TESTED_NO_BROWSER
NORMAL_OPENHANDS_UI=PASS|FAIL|NOT_TESTED_NO_BROWSER
ROUTE_NAVIGATION_5X=PASS|FAIL|NOT_TESTED_NO_BROWSER
TAB_BACKGROUND_RETURN_5X=PASS|FAIL|NOT_TESTED_NO_BROWSER
IDLE_FOCUS_RETURN=PASS|FAIL|NOT_TESTED_NO_BROWSER
FULL_DOCUMENT_RELOAD_OBSERVED=YES|NO|NOT_TESTED_NO_BROWSER
ACTIVE_BACKEND_CONTEXT_ERROR=YES|NO|NOT_TESTED_NO_BROWSER
REMOTE_WRITE_EXECUTED_BY_PATCH=NO
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires ALL of these:

- production frontend remains non-HMR;
- public automation API works;
- normal push is isolated;
- sync write path is isolated;
- initial push remains isolated;
- no live `/opt/TBA` Git write cwd;
- backend validation passes;
- no remote write executed by the patch.
