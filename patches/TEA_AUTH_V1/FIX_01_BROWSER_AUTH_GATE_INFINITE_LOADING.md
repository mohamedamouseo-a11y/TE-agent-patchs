# TEA_AUTH_V1 — FIX 01 — Browser Auth Gate Infinite Loading

## Scope
Target NEW TEA only:
- `/opt/TBA`
- `https://tea.tamiyouz.com`

This is a runtime repair for the current production symptom:

> Opening a protected/deep-link route such as `/settings/agents` leaves the browser on an endless black/loading screen instead of showing the TEA login screen or the authenticated application.

The local auth backend already exists and MUST be preserved:
- SQLite user store
- Argon2id password hashing
- `/api/auth/login`
- `/api/auth/logout`
- `/api/auth/me`
- `/api/auth/change-password`
- `/api/auth/activate`
- secure HttpOnly session cookie
- server-side SUPER_ADMIN authorization

Do NOT rebuild auth from scratch.

---

## Hard constraints

- Preserve current local-mode machine/API-key authentication for agent/internal calls.
- Preserve `/api/automation`.
- Preserve Developer Hub server-side authorization.
- Preserve production no-HMR architecture.
- No client-side secrets.
- Do not print existing API keys, admin keys, passwords, session secrets, activation tokens, password hashes, or cookies.
- No Git push/pull/fetch/commit or other remote write.
- Do not rotate credentials in this fix.
- Do not modify unrelated TEA screens.

---

## 1. Diagnose the actual hang BEFORE editing

Inspect production runtime and identify exactly which state never resolves.

Check all of the following:

### Auth service
Verify `tea-auth.service` is active and listening on its expected localhost port.

### Nginx auth proxy
Verify same-origin requests to these routes are routed to the auth service and terminate quickly:

- `GET /api/auth/me`
- `POST /api/auth/login`
- `POST /api/auth/logout`
- `POST /api/auth/change-password`
- `POST /api/auth/activate`

`GET https://tea.tamiyouz.com/api/auth/me` while unauthenticated MUST return a finite unauthenticated response (normally HTTP 401) quickly. It MUST NOT hang, redirect-loop, return the SPA HTML, or proxy to the Agent Server.

Record:
- response status
- content-type
- elapsed time
- nginx upstream selected

Do not print cookies or secrets.

### Browser/root gate
Inspect the production source currently deployed on `/opt/TBA`, especially:
- `src/root.tsx`
- `src/components/features/backends/tea-login-screen.tsx`
- any new auth context/hook/service added by TEA_AUTH_V1

Identify whether the infinite loading is caused by one or more of:
- `/api/auth/me` request never completing
- incorrect nginx route order
- request missing `credentials: "include"`
- treating HTTP 401 as pending/error forever
- React Query retrying 401 indefinitely
- auth query enabled only after another bootstrap query that itself requires auth
- old API-key `/server_info` / `/api/settings` bootstrap running before human auth resolution
- lazy component import failure hidden behind spinner
- auth state initialized as loading and never transitioned on failure
- redirect loop
- session cookie path/domain mismatch

Do not guess. State the concrete root cause in the final report.

---

## 2. Required browser auth state machine

Human browser authentication MUST resolve before protected TEA bootstrap queries.

Implement one deterministic gate with these states:

```text
BOOT
  -> GET /api/auth/me

PENDING
  -> bounded loading state only

401 / unauthenticated
  -> TeaLoginScreen

200 + must_change_password=true
  -> ChangePassword screen

200 + authenticated user
  -> Existing TEA application/bootstrap

network/server failure
  -> visible recoverable Auth Error screen + Retry
```

### Critical rule
An HTTP 401 from `/api/auth/me` is a normal unauthenticated state.
It MUST NOT be treated as an endless loading state.

### Loading bound
The initial auth gate may show loading briefly, but it must never remain indefinitely.
Use an explicit request timeout/abort (reasonable target: 5–10 seconds maximum) and transition to a recoverable error state.

Do not use timeout as a substitute for fixing a broken nginx/backend route; first fix the actual route/service problem.

---

## 3. Root ordering

For this TEA deployment, the browser user auth gate must happen BEFORE protected application bootstrap that can otherwise wait on machine API-key/backend state.

Expected order:

```text
Human session check (/api/auth/me)
        |
        +-- unauthenticated -> TeaLoginScreen
        |
        +-- password change required -> ChangePassword
        |
        +-- authenticated -> continue existing TEA bootstrap
                                 |
                                 +-- existing backend/config/server_info logic
```

Do NOT delete the existing API-key machine authentication logic.
Keep it for backend/agent communications after human browser auth succeeds.

Do NOT let these old gates hide the new human login behind an infinite spinner:
- onboarding
- `/server_info`
- `/api/settings`
- backend health
- main-app cookie auth
- API-key entry gate

For this local TEA human login deployment, unauthenticated browser users must see `TeaLoginScreen`, not `ApiKeyEntryScreen`.

---

## 4. Auth client requirements

All browser session auth requests must be same-origin and use cookies correctly.

Required behavior:

```ts
fetch('/api/auth/me', {
  credentials: 'include',
  ...
})
```

Apply equivalent cookie inclusion to login/logout/change-password/activate where relevant.

If React Query is used:
- `retry: false` for 401/auth verdicts
- do not translate 401 to infinite pending
- distinguish `isPending` from unauthenticated result
- invalidate/refetch `/api/auth/me` immediately after login/logout/password change
- do not clear a valid authenticated user into a spinner on window-focus background refetch

---

## 5. Deep links

Unauthenticated navigation directly to:

`https://tea.tamiyouz.com/settings/agents`

must show the TEA Login screen while preserving the intended destination.

After successful login and any required password change, return the user to the original safe same-origin destination.

Prevent open redirects:
- only relative/same-origin TEA paths may be restored
- reject external redirect targets

---

## 6. Login screen

Use the premium TEA login screen already created in Phase 01.
Do not redesign it in this fix.

It must authenticate using real:

`POST /api/auth/login`

with email + password.

On successful login:
1. session cookie is received
2. `/api/auth/me` is refreshed
3. auth gate resolves
4. if `must_change_password=true`, show change-password flow
5. otherwise continue into TEA

On invalid credentials:
- show a clear inline error
- no full-page black spinner
- no leaking whether another account exists

---

## 7. Production routing validation

Nginx must not allow SPA fallback to swallow `/api/auth/*`.

Ensure API locations have precedence over frontend static fallback.

Verify:
- `/api/auth/me` -> auth service
- `/api/automation/*` -> existing automation backend
- normal app routes -> production static SPA

No Vite dev server / HMR client / HMR websocket.

---

## 8. Required runtime tests

Perform tests against production after rebuild/reload.

### Unauthenticated/incognito
Open directly:
- `/`
- `/settings/agents`
- `/developer-hub`

Expected:
- Login screen renders promptly (target <= 3 seconds under normal local-server conditions)
- no endless black/loading screen
- no API-key entry screen for human login
- no redirect loop

### `/api/auth/me`
Unauthenticated:
- finite response
- expected unauthenticated verdict
- no retry storm

Authenticated session test (use existing server-side test capability without printing credentials):
- `/api/auth/me` -> authenticated identity
- protected app proceeds
- SUPER_ADMIN Developer Hub authorization remains server-side

### Failure test
Temporarily simulate an auth service fetch failure in a controlled/non-destructive way or validate the client error path:
- visible error state
- Retry control works
- never infinite loading

### Regression
- `/api/automation` PASS
- Developer Hub PASS for SUPER_ADMIN
- non-SUPER_ADMIN remains blocked
- machine API-key auth preserved
- production build PASS
- no HMR
- no client secret exposure

---

## 9. Production rebuild

After source changes:
- run parse/type/build checks appropriate to the project
- rebuild the production static frontend
- reload only services actually changed
- verify public `https://tea.tamiyouz.com`

Do not consider HTTP 200 alone sufficient. Verify visible runtime behavior.

---

## Completion report

Return ONLY this report, with no secrets:

```text
FIX=TEA_AUTH_V1_FIX_01_BROWSER_AUTH_GATE_INFINITE_LOADING
ROOT_CAUSE=
AUTH_SERVICE=PASS|FAIL
AUTH_ME_UNAUTH_STATUS=
AUTH_ME_UNAUTH_TIME_MS=
AUTH_ME_ROUTE_TARGET=AUTH_SERVICE|WRONG_UPSTREAM|SPA|UNKNOWN
AUTH_ME_401_RESOLVES_TO_LOGIN=PASS|FAIL
AUTH_FETCH_CREDENTIALS_INCLUDE=PASS|FAIL
AUTH_QUERY_401_RETRY_DISABLED=PASS|FAIL
AUTH_REQUEST_TIMEOUT=PASS|FAIL
AUTH_ERROR_FALLBACK=PASS|FAIL
ROOT_AUTH_GATE_BEFORE_BACKEND_BOOTSTRAP=PASS|FAIL
OLD_API_KEY_MACHINE_AUTH_PRESERVED=PASS|FAIL
TEA_LOGIN_SCREEN_RENDER=PASS|FAIL
ROOT_INCOGNITO_LOGIN=PASS|FAIL
SETTINGS_AGENTS_INCOGNITO_LOGIN=PASS|FAIL
DEVELOPER_HUB_INCOGNITO_AUTH_GATE=PASS|FAIL
DEEP_LINK_RETURN_AFTER_AUTH=PASS|FAIL|NOT_TESTED
MUST_CHANGE_PASSWORD_GATE=PASS|FAIL|NOT_TESTED
ENDLESS_LOADING_GONE=YES|NO
BLACK_SCREEN_GONE=YES|NO
API_AUTOMATION=PASS|FAIL
SUPERADMIN_DEVELOPER_HUB=PASS|FAIL|NOT_TESTED
NON_SUPERADMIN_BLOCKED=PASS|FAIL|NOT_TESTED
PRODUCTION_BUILD=PASS|FAIL
PRODUCTION_REBUILD=PASS|FAIL
VITE_HMR_PRESENT=YES|NO
CLIENT_SECRET_EXPOSURE=YES|NO
REMOTE_WRITE_EXECUTED=NO
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` is allowed only when `ENDLESS_LOADING_GONE=YES` and the unauthenticated browser visibly renders the TEA login screen in production.
