# FIX 29 — Restore Agent Server `/server_info` while keeping static no-HMR production

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed production symptom

After FIX 28, the browser opens **Manage backends** and shows the Local backend disconnected with:

`Could not determine this backend's agent-server version. Agent Canvas requires agent-server 1.28.0 or newer, but this backend did not return a valid version from /server_info.`

This appeared after production was moved from the project's integrated launcher to a custom `sirv-cli` static frontend runtime.

## Source evidence from TEA

The repository's existing `scripts/dev-with-automation.mjs` already supports a **static production mode** and integrated ingress routing.

Its intended routing includes:

- `/api/automation/*` → Automation backend
- `/api/*` → Agent Server
- `/sockets` → Agent Server
- `/server_info` → Agent Server
- `/health`, `/ready`, `/alive`, `/docs`, `/redoc` → Agent Server
- all other routes → frontend

The same launcher supports `--static`, `--static-dir`, and `--skip-build`, and waits for Agent Server `/server_info` before declaring the stack ready.

Therefore the production architecture must preserve the **agent-server**, not only the static frontend and automation backend.

## Goal

Restore a valid Agent Server behind `tea.tamiyouz.com` without reintroducing Vite/HMR and without breaking the FIX 28 Git-isolation work.

Do not execute any Developer Hub Push/Pull/Sync while applying this patch.

---

## Part A — Diagnose current runtime exactly

Inspect read-only first:

1. `tba-openhands.service` exact ExecStart/environment.
2. process tree and listening ports.
3. current custom `scripts/prod-with-automation.mjs` behavior.
4. current ingress on port 8001.
5. HTTP response from:
   - `https://tea.tamiyouz.com/server_info`
   - local ingress `/server_info`
   - any currently running Agent Server port if present.
6. Determine whether `/server_info` is currently:
   - served by `sirv` as SPA HTML,
   - 404/502,
   - or routed to the wrong backend.
7. Determine whether an Agent Server process is running at all.

Record exact root cause before changing runtime.

---

## Part B — Prefer the project's integrated static launcher

Do NOT return to `react-router dev` or Vite.

Prefer reusing the existing, repository-supported launcher instead of maintaining a partial custom production proxy.

Target architecture should be equivalent to:

- frontend: **static production build**
- Agent Server: running and version-compatible
- Automation backend: running
- ingress: routes all required Agent Server and Automation paths correctly
- no Vite/HMR client or websocket

The project's own `scripts/dev-with-automation.mjs --static` path already implements this architecture. Reuse it directly, or make the smallest safe production wrapper around the same exported logic if service constraints require it.

### Required route contract

Public `tea.tamiyouz.com` must route:

```text
/api/automation/*  -> automation backend
/api/*             -> agent-server
/sockets            -> agent-server
/server_info        -> agent-server
/health             -> agent-server
/ready              -> agent-server
/alive              -> agent-server
/docs               -> agent-server
/redoc              -> agent-server
/*                  -> static production frontend
```

Do not let SPA fallback swallow any backend path.

### Agent Server requirements

1. Use the currently intended OpenHands Agent Server source/version from project config/environment.
2. It must return HTTP 200 JSON from `/server_info`.
3. The JSON must contain a valid agent-server version accepted by Agent Canvas (>= 1.28.0).
4. Do not fake or hardcode a version response in nginx/frontend.
5. Restore the actual backend service.
6. Preserve shared session/API-key behavior already used by the stack; never print secret values.

### Production frontend requirements

1. Keep serving `/opt/TBA/build` (or the validated production build output).
2. `VITE_HMR_PRESENT=NO`.
3. no `/@vite/client`.
4. no Vite HMR websocket.
5. no `react-router dev` production process.

### Automation requirements

Preserve FIX 28:

- `/api/automation` public PASS
- GitHub connection API PASS
- Developer Hub repo/status/review APIs PASS
- local automation source/package fix remains valid

---

## Part C — Preserve all Developer Hub safety fixes

Do not regress any of these:

- FIX 23 commit timeout / hook process cleanup
- FIX 24 isolated initial push
- FIX 25 fully-qualified refspec
- FIX 26 authoritative remote verification
- FIX 28 normal push isolation
- FIX 28 sync write isolation

All Git-mutating Developer Hub actions must remain isolated from `/opt/TBA` live working tree.

During this FIX task:

- no Developer Hub Execute
- no git push/pull/fetch
- no remote write
- no reset/clean/stash/checkout of live tree
- no printing tokens/secrets

Do not modify the existing 37 staged entries merely to complete this runtime repair.

---

## Part D — Runtime validation

After repair, verify from the actual public domain:

1. `/` HTTP 200 and UI loads.
2. `/server_info` HTTP 200, JSON content type, valid Agent Server version >=1.28.0.
3. `/api/automation/...` read-only status endpoint works.
4. Developer Hub read-only repo/status API works.
5. Agent Canvas Local backend no longer reports `Could not determine this backend's agent-server version`.
6. If browser session exists, Local backend becomes Connected and normal OpenHands UI loads.
7. no Vite HMR client/websocket.
8. no full reload caused by HMR.

If no browser session is available, do not fabricate UI PASS; report browser values as `NOT_TESTED_NO_BROWSER`.

---

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_29_RESTORE_AGENT_SERVER_SERVER_INFO_WITH_STATIC_PRODUCTION
CURRENT_RUNTIME_BEFORE=
SERVER_INFO_BEFORE_HTTP=
SERVER_INFO_BEFORE_CONTENT_TYPE=
AGENT_SERVER_PROCESS_BEFORE=RUNNING|NOT_RUNNING
ROOT_CAUSE=
RUNTIME_AFTER=
AGENT_SERVER_PROCESS_AFTER=RUNNING|NOT_RUNNING
AGENT_SERVER_INTERNAL_PORT=
AGENT_SERVER_VERSION=
AGENT_SERVER_VERSION_COMPATIBLE=YES|NO
SERVER_INFO_PUBLIC_HTTP=
SERVER_INFO_PUBLIC_JSON=PASS|FAIL
SERVER_INFO_PUBLIC_VERSION_VALID=PASS|FAIL
AGENT_SERVER_API_ROUTING=PASS|FAIL
AGENT_SERVER_SOCKET_ROUTING=PASS|FAIL
API_AUTOMATION_PUBLIC=PASS|FAIL
GITHUB_CONNECTION_API=PASS|FAIL
REPO_STATUS_API=PASS|FAIL
HOME_UI=PASS|FAIL
DEVELOPER_HUB_UI=PASS|FAIL|NOT_TESTED_NO_BROWSER
LOCAL_BACKEND_CONNECTED=PASS|FAIL|NOT_TESTED_NO_BROWSER
VERSION_ERROR_GONE=YES|NO|NOT_TESTED_NO_BROWSER
VITE_HMR_PRESENT=YES|NO
VITE_HMR_WEBSOCKET=YES|NO
INITIAL_PUSH_ISOLATED=PASS|FAIL
NORMAL_PUSH_ISOLATED=PASS|FAIL
SYNC_WRITE_ISOLATED=PASS|FAIL
LIVE_TBA_USED_AS_GIT_WRITE_CWD=YES|NO
LIVE_STAGED_COUNT_BEFORE=
LIVE_STAGED_COUNT_AFTER=
LIVE_WORKTREE_BYTES_PRESERVED=YES|NO
REMOTE_WRITE_EXECUTED_BY_PATCH=NO
TOKEN_SECRET_SAFE=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
BUILD_VALIDATION=PASS|FAIL|NOT_CHANGED
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires a real compatible Agent Server `/server_info`, preserved static no-HMR frontend, preserved automation API, and preserved Git isolation.
