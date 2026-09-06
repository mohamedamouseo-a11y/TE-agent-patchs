# FIX 32 — Developer Hub background refetch without skeleton flicker

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed symptom from user video

The page does not perform a full document reload anymore. Instead, when the browser/tab loses focus and the user returns, Developer Hub data (especially Repository Status) temporarily disappears and is replaced by skeleton/loading UI for several seconds, then the data reappears.

This is consistent with a focus/visibility-triggered refetch or revalidation that incorrectly reuses the initial-loading UI state for a background refresh.

## Goal

Keep the last successful Developer Hub data visible during background refetch/revalidation. Skeletons must be used only for first load when no cached/successful data exists.

Do not change production runtime/HMR architecture. Do not weaken auth or API safety.

## Diagnose first

Inspect the actual Developer Hub frontend data flow, including:
- route/component for `/developer-hub`
- `github-panel.tsx` and related Developer Hub components
- `developer-hub-service.api.ts`
- React Query/useQuery/query client configuration if used
- any `focus`, `visibilitychange`, polling, interval, route revalidation or manual refresh listeners
- any state transitions that set repository/status data to undefined/null before refetch completes

Record the exact trigger and exact condition that causes skeleton rendering.

## Required behavior

1. Initial page load with no data:
   - skeleton/loading UI is allowed.

2. Background refetch after prior successful data:
   - keep prior Repository Status and other existing cards visible;
   - do not blank the data;
   - do not replace the whole card with skeletons;
   - optional small `Refreshing...` indicator is allowed, but layout must remain stable.

3. Focus/visibility behavior:
   - if refetch-on-focus is not required for correctness, disable it for Developer Hub queries;
   - if it is required, keep it as a background fetch with stale data preserved;
   - do not globally disable useful refetch behavior for unrelated OpenHands features.

4. Query/cache rules when React Query is used:
   - distinguish `isLoading`/`isPending` from `isFetching`;
   - render initial skeleton only when there is no usable data;
   - use cached/previous data during refetch;
   - do not clear query data at refetch start;
   - use an appropriate non-zero stale/cache strategy only if needed and scoped to Developer Hub.

5. Manual refresh buttons must still work and must also preserve current data while refreshing.

6. Errors during background refresh:
   - preserve last-known-good data;
   - show a non-destructive error/refresh indicator;
   - do not blank the screen unless there has never been successful data.

## Preserve

- FIX27+ production non-HMR runtime
- FIX30 server-side authorization
- FIX31 runtime hook fix
- `/api/automation`
- Super Admin gate
- token/secret safety
- isolated Git write architecture
- no remote Git write during this patch

## Restrictions

Do NOT:
- switch back to Vite/HMR
- Execute Reviewed Action
- git push/pull/fetch
- perform any remote write
- print token/key/secret
- mask a real full-document reload if one is still detected; report it separately

## Validation

After code changes, rebuild production and validate actual UI if browser session is available.

Required tests:
1. open `/developer-hub` and wait for loaded Repository Status;
2. background tab for 10–20 seconds, return, repeat 5 times;
3. navigate to another OpenHands route and back 5 times;
4. trigger any existing automatic/manual refresh;
5. confirm existing data remains visible throughout background refetch;
6. confirm skeleton appears only on true first load with no data;
7. confirm no full document reload;
8. confirm no HMR client/websocket;
9. confirm `/api/automation` remains healthy.

If no browser session exists, do not claim UX PASS; return NOT_TESTED_NO_BROWSER for visual fields.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_32_BACKGROUND_REFETCH_NO_SKELETON_FLICKER
ROOT_CAUSE_TRIGGER=
ROOT_CAUSE_LOADING_CONDITION=
DATA_FETCH_LIBRARY=
REFETCH_ON_WINDOW_FOCUS_BEFORE=
REFETCH_ON_WINDOW_FOCUS_AFTER=
INITIAL_LOADING_DISTINGUISHED_FROM_BACKGROUND_FETCH=PASS|FAIL
PREVIOUS_DATA_PRESERVED_DURING_REFETCH=PASS|FAIL
QUERY_DATA_CLEARED_ON_REFETCH=YES|NO
BACKGROUND_REFRESH_ERROR_PRESERVES_DATA=PASS|FAIL
MANUAL_REFRESH_PRESERVES_DATA=PASS|FAIL
REPOSITORY_STATUS_SKELETON_FIRST_LOAD_ONLY=PASS|FAIL
TAB_BACKGROUND_RETURN_5X=PASS|FAIL|NOT_TESTED_NO_BROWSER
ROUTE_NAVIGATION_5X=PASS|FAIL|NOT_TESTED_NO_BROWSER
FULL_DOCUMENT_RELOAD_OBSERVED=YES|NO|NOT_TESTED_NO_BROWSER
VISUAL_FLICKER_GONE=YES|NO|NOT_TESTED_NO_BROWSER
API_AUTOMATION=PASS|FAIL
SERVER_SIDE_AUTH_PRESERVED=PASS|FAIL
VITE_HMR_PRESENT=YES|NO
HMR_WEBSOCKET_PRESENT=YES|NO
TOKEN_SECRET_SAFE=PASS|FAIL
BUILD_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL|NOT_CHANGED
REMOTE_WRITE_EXECUTED=NO
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` requires the loaded data to remain visible during background refetch and no production HMR/full reload regression.
