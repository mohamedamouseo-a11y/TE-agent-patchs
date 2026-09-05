# FIX 03 — Developer Hub stuck loading

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com/developer-hub`).

Observed production symptom: the Developer Hub page renders, the GitHub panel renders, but the Repository Status card remains in skeleton/loading state indefinitely.

Do not guess the cause. Diagnose the exact request/hook that never settles, then apply the smallest fix.

## Required diagnosis

Inspect the current live frontend implementation and identify the exact query used by the Repository Status card, including:

- component path
- hook/query path
- API service method
- final production request URL
- auth/header behavior
- HTTP result or pending/error state

Verify the request through the production domain, not direct Vite only.

Expected Developer Hub backend prefix is the current production prefix:

`/api/automation/v1/developer-hub`

Confirm the status request resolves to the intended status endpoint and is not using an empty URL, stale path, wrong baseURL, wrong auth client, or an unresolved Promise.

If the backend endpoint itself hangs, identify the blocking operation and make the status endpoint bounded/non-blocking. Repository status must not perform an unbounded network/Git operation during page render.

## Fix requirements

1. Repository Status loading must always terminate into one of:
   - loaded data
   - explicit error state with Retry

2. No infinite skeleton is allowed.

3. Use the existing TEA API/auth client conventions. Do not introduce a new client, framework, service, port, proxy, or dependency.

4. Add a reasonable request timeout for Developer Hub read/status queries. A timeout must become a visible recoverable error, not permanent loading.

5. Keep Developer Hub cards independent: failure of Repository Status must not freeze GitHub, Operations, Audit, Push Control, MCP, AI Context, or Source Export.

6. Do not weaken Super Admin authorization.

7. Do not change Git behavior, GitHub credentials, review/execute logic, nginx, ports, or unrelated OpenHands functionality unless the proven root cause is an incorrect Developer Hub route binding already present in the current implementation.

8. Do not perform Git writes while applying this fix.

## Verification

After the fix, restart only NEW TEA services if required and verify in a clean browser session:

- `https://tea.tamiyouz.com/developer-hub` loads.
- Repository Status skeleton disappears within a bounded time.
- Repository Status shows real data when backend succeeds.
- Simulated/forced read failure shows an error + Retry instead of infinite skeleton.
- GitHub interactive module still works.
- Super Admin gate still passes.
- normal OpenHands UI still works.
- restart survival passes.

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_03_LOADING_STATE
ROOT_CAUSE=
STATUS_COMPONENT=
STATUS_HOOK=
STATUS_API_METHOD=
STATUS_REQUEST_URL=
STATUS_HTTP_RESULT=
INFINITE_SKELETON_GONE=PASS|FAIL
STATUS_DATA_LOADS=PASS|FAIL
ERROR_STATE_RETRY=PASS|FAIL
OTHER_DEVHUB_CARDS_UNBLOCKED=PASS|FAIL
GITHUB_MODULE=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
DOMAIN_DEVHUB_UI=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
RESTART_SURVIVAL=PASS|FAIL
CHANGED_FILES=<paths>
STATUS=READY|FAILED
```