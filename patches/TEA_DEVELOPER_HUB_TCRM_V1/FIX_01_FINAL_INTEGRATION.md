# FIX 01 — Final Developer Hub Integration

Apply only to the new TEA OpenHands source tree. Do not perform Git writes.

## Confirmed failures

1. Frontend API service calls use empty request paths. Bind every Developer Hub call to the existing backend prefix `/api/automation/v1/developer-hub` using the project's current API client/auth conventions.

2. Add a dedicated `Developer Hub` navigation entry pointing to `/developer-hub`. It must be visible only to Super Admin. Do not reuse the Automations navigation item.

3. Keep production ingress and the active automation backend consistent. `/api/automation` must route to the active automation backend currently intended to run on port `18003`. Do not create another service or port.

4. Local-mode `canManage: true` is not sufficient authorization. Developer Hub must enforce a server-side Super Admin boundary on every endpoint. Anonymous 401 is not proof of Super Admin-only access.

Use the existing OpenHands identity/session/permission architecture where possible. If the current build has no authenticated non-admin role distinction, do not fake the verification result; implement the smallest Developer-Hub-only admin gate supported by the existing authentication/config model, deny by default, and keep credentials server-side.

## Verification

Verify through the production domain, not direct Vite only:

- `/developer-hub` renders and frontend API calls resolve.
- Developer Hub navigation exists and is hidden for non-Super-Admin sessions.
- Authenticated Super Admin can access the Developer Hub API.
- Authenticated non-Super-Admin receives 403.
- `/api/automation/v1/developer-hub/status` works through production ingress.
- Active automation backend is port `18003` and `/api/automation` routes to it.
- Normal OpenHands home and conversation UI still work.
- No new framework, service, dependency, or port is introduced.

Restart only the new TEA services if required.

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_01
FRONTEND_API_PATHS=PASS|FAIL
DEVHUB_NAV=PASS|FAIL
NAV_SUPERADMIN_ONLY=PASS|FAIL
SUPERADMIN_API=PASS|FAIL
NON_SUPERADMIN_403=PASS|FAIL
AUTOMATION_PORT=18003|OTHER
INGRESS_API_AUTOMATION=PASS|FAIL
DOMAIN_DEVHUB_API=PASS|FAIL
DOMAIN_DEVHUB_UI=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=<paths>
STATUS=READY|FAILED
```