# FIX 29 — Restore Developer Hub in production build without reintroducing HMR

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed regression

After moving production off Vite/HMR, the live OpenHands UI loads, but the **Developer Hub sidebar entry is missing** and the user cannot access the custom Developer Hub from the UI.

Do not treat a healthy `/settings/agents` page as sufficient. Developer Hub must be restored as part of production parity.

## Important source fact

The current GitHub `mohamedamouseo-a11y/TEA` `main` snapshot does not expose `src/routes/developer-hub.tsx` through the contents API, while the live pre-production-switch system previously had working Developer Hub source and UI. Therefore first prove whether the current `/opt/TBA` working tree still contains the custom Developer Hub files and route/sidebar wiring, or whether the production build was generated from a source state that omitted them.

Do not overwrite custom Developer Hub code from stock OpenHands.

## Goal

Restore the exact existing/custom Developer Hub to production while preserving:

- production static/non-HMR frontend
- `/api/automation`
- GitHub token persistence
- Developer Hub backend router/API
- FIX20–FIX28 safety behavior
- no remote Git writes

## Inspect first

Read-only inspect all of:

- `/opt/TBA/src/routes/developer-hub.tsx`
- `/opt/TBA/src/components/features/developer-hub/`
- `/opt/TBA/src/root.tsx`
- route registration/config files
- sidebar/navigation source that previously rendered `Developer Hub`
- current `build/` artifact
- current `tba-openhands.service`
- current production launcher
- `/api/automation` runtime

Determine separately:

1. `DEVELOPER_HUB_SOURCE_PRESENT`
2. `DEVELOPER_HUB_ROUTE_REGISTERED`
3. `DEVELOPER_HUB_SIDEBAR_ENTRY_PRESENT`
4. `DEVELOPER_HUB_PRESENT_IN_BUILD`
5. direct HTTP behavior of `/developer-hub`

## Recovery rules

### Case A — source exists but production build omitted it

If source + route + sidebar wiring are present in `/opt/TBA`, rebuild production from that exact working tree and redeploy atomically. Do not alter Developer Hub code unnecessarily.

### Case B — route/sidebar wiring is missing but Developer Hub source still exists

Restore only the missing route/sidebar registration using the existing local Developer Hub implementation as source of truth. Preserve Super Admin gating and existing route behavior.

### Case C — Developer Hub source files themselves are missing

Recover from the safest available local source first:

- non-destructive local backups
- previous build/source copies
- patch history / earlier known-good local files

Do not replace the custom module with stock OpenHands or an empty placeholder.

If exact recovery is impossible, return FAILED with blocker instead of fabricating a new Developer Hub.

## Production requirements

After repair:

- `/developer-hub` must load directly
- sidebar must visibly contain `Developer Hub`
- navigating to it must not full-reload the document
- `/settings/agents` and normal OpenHands routes remain healthy
- frontend remains production-built, not Vite dev
- `/@vite/client` absent
- no HMR websocket
- `/api/automation` works

## Strict restrictions

Do NOT:

- git push/pull/fetch
- Execute Reviewed Action
- create any remote commit/branch
- print token/secret
- switch production back to Vite/react-router dev
- delete or replace custom Developer Hub with stock UI

Local source/build/service changes required to restore the route are allowed.

## Validation

Run at minimum:

- parse/typecheck/build as applicable
- production rebuild
- service restart only after successful build
- HTTP check `/`
- HTTP check `/settings/agents`
- HTTP check `/developer-hub`
- API check `/api/automation`
- inspect production HTML/network for no Vite HMR

If a browser is available, verify the sidebar entry is actually visible and click it.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_29_RESTORE_DEVELOPER_HUB_IN_PRODUCTION_BUILD
ROOT_CAUSE=
DEVELOPER_HUB_SOURCE_PRESENT_BEFORE=YES|NO
DEVELOPER_HUB_ROUTE_REGISTERED_BEFORE=YES|NO
DEVELOPER_HUB_SIDEBAR_ENTRY_PRESENT_BEFORE=YES|NO
DEVELOPER_HUB_PRESENT_IN_BUILD_BEFORE=YES|NO
RECOVERY_SOURCE=
DEVELOPER_HUB_SOURCE_PRESENT_AFTER=YES|NO
DEVELOPER_HUB_ROUTE_REGISTERED_AFTER=PASS|FAIL
DEVELOPER_HUB_SIDEBAR_ENTRY_AFTER=PASS|FAIL
DEVELOPER_HUB_PRESENT_IN_BUILD_AFTER=PASS|FAIL
DEVELOPER_HUB_HTTP=PASS|FAIL
DEVELOPER_HUB_UI=PASS|FAIL|NOT_TESTED_NO_BROWSER
SETTINGS_AGENTS_UI=PASS|FAIL
HOME_UI=PASS|FAIL
API_AUTOMATION=PASS|FAIL
PRODUCTION_FRONTEND_RUNTIME=
VITE_HMR_PRESENT=YES|NO
HMR_WEBSOCKET_PRESENT=YES|NO
FULL_RELOAD_ON_DEVELOPER_HUB_NAV=YES|NO|NOT_TESTED_NO_BROWSER
SUPERADMIN_GATE=PASS|FAIL
TOKEN_SECRET_SAFE=PASS|FAIL
BUILD_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL|NOT_CHANGED
REMOTE_WRITE_EXECUTED=NO
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires the Developer Hub sidebar entry and `/developer-hub` route to be restored in the actual production build while production remains non-HMR.
