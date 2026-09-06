# FIX 31 — Developer Hub `useState is not defined` production runtime recovery

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed live failure

The production `/developer-hub` route currently renders:

```text
An error occurred
useState is not defined
```

This appeared after FIX30 rebuilt the production frontend with server-side Developer Hub auth gating.

Do not treat HTTP 200 as success; the actual route UI must render without runtime errors.

## Goal

Find the exact source/bundle origin of the bare `useState` reference, repair it minimally, rebuild production, and prove `/developer-hub` works while preserving all FIX20–FIX30 behavior.

## Diagnose first — no guessing

Inspect the current local source and production bundle. Search all Developer Hub / auth-gating changes made by FIX29/FIX30, including at minimum:

- `/opt/TBA/src/routes/developer-hub.tsx`
- `/opt/TBA/src/components/features/developer-hub/**`
- `/opt/TBA/src/components/features/sidebar/sidebar-rail-body.tsx`
- `/opt/TBA/src/api/developer-hub-service.api.ts`
- `/opt/TBA/src/root.tsx`
- route configuration
- generated `/opt/TBA/build/**` JavaScript related to `/developer-hub`

Find the exact file/line where `useState` is referenced without being in scope.

Possible causes include, but are not limited to:
- missing `import { useState } from "react"`;
- refactor removed the React hook import while leaving calls;
- code uses bare `useState` but imports only `React` or other hooks;
- production bundling exposes a path not exercised by prior build/type validation.

Record the exact root cause. Do not blindly add imports to multiple files.

## Repair rules

1. Make the smallest correct source change at the actual offending file.
2. Prefer a normal React hook import consistent with project style.
3. Do not suppress runtime errors with globals/fallback shims.
4. Do not edit generated build assets by hand as the primary fix.
5. Rebuild production from source after the fix.
6. Preserve server-side Developer Hub authorization from FIX30.
7. Preserve production static/non-HMR runtime.
8. Preserve `/api/automation` routing and Developer Hub backend.
9. Preserve all Git isolation, remote verification, token secrecy and Super Admin gates.

## Validation

Run at minimum:

- source syntax/parse validation;
- `npm run typecheck`;
- `npm run build`;
- deploy the new production build atomically;
- restart/reload only the required frontend service;
- verify `/`;
- verify `/settings/agents`;
- verify `/developer-hub` actual rendered UI, not only HTTP status;
- verify `/api/automation` remains working;
- verify no `/@vite/client` and no HMR websocket.

If no browser session exists, use production JS/runtime logs or a headless/browser-capable mechanism if already available. Do not claim UI PASS from HTTP 200 alone. If true rendered verification is impossible, return `DEVELOPER_HUB_UI=NOT_TESTED_NO_BROWSER` and STATUS=FAILED.

## Strict restrictions

Do NOT:
- Execute Reviewed Action;
- git push/pull/fetch;
- create a TEA source remote commit;
- perform any remote Git write;
- switch production back to Vite/react-router dev;
- expose or print any token/key/secret;
- weaken FIX30 server-side authorization.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_31_DEVELOPER_HUB_USESTATE_RUNTIME_ERROR
ROOT_CAUSE_FILE=
ROOT_CAUSE_LINE=
ROOT_CAUSE=
USESTATE_REFERENCE_BEFORE=
USESTATE_IMPORT_BEFORE=
REPAIR=
USESTATE_SCOPE_AFTER=PASS|FAIL
PARSE_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL
BUILD_VALIDATION=PASS|FAIL
PRODUCTION_REBUILD=PASS|FAIL
HOME_UI=PASS|FAIL
SETTINGS_AGENTS_UI=PASS|FAIL
DEVELOPER_HUB_HTTP=PASS|FAIL
DEVELOPER_HUB_UI=PASS|FAIL|NOT_TESTED_NO_BROWSER
USESTATE_RUNTIME_ERROR_GONE=YES|NO|NOT_TESTED_NO_BROWSER
API_AUTOMATION=PASS|FAIL
SERVER_SIDE_AUTH_PRESERVED=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
NON_SUPERADMIN_BLOCKED=PASS|FAIL|NOT_TESTED
VITE_HMR_PRESENT=YES|NO
HMR_WEBSOCKET_PRESENT=YES|NO
TOKEN_SECRET_SAFE=PASS|FAIL
REMOTE_WRITE_EXECUTED=NO
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires the actual `/developer-hub` UI to render without `useState is not defined`, production to remain non-HMR, `/api/automation` to work, and FIX30 server-side authorization to remain intact.
