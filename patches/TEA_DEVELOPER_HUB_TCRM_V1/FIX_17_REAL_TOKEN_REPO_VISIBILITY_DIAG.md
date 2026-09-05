# FIX 17 — Real Token Repository Visibility Diagnostic

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed production state

After FIX 16, the Super Admin re-entered the PAT and production now visibly shows an authenticated GitHub login:
`mohamedamouseo-a11y`

But Repository still shows:
`No repositories`

Therefore token authentication is now working, but repository visibility is still unresolved.

Do not guess whether this is PAT scope/access or a TEA backend/frontend bug. Prove it from the live stored token without printing the token.

## Task

1. Do NOT modify code initially.
2. Using the persisted token internally, call GitHub REST read-only and record sanitized results only:
   - `GET /user`
   - `GET /user/repos?per_page=100&page=1&sort=updated&affiliation=owner,collaborator,organization_member`
3. If pagination is needed, continue until the last short page.
4. Record:
   - HTTP status for `/user`
   - authenticated login
   - raw repository count from GitHub before any TEA filtering
   - whether any repo named exactly `TEA` appears
   - exact `full_name` of `TEA` if present
   - sanitized permission metadata only (`permissions`, `role_name`, `visibility`, `private`, `default_branch`)
5. Compare that raw GitHub result against:
   - TEA backend repository endpoint count
   - production UI repository count

## Root cause classification

Choose exactly one:
- `TOKEN_NO_REPO_ACCESS`: GitHub authenticates but returns zero repositories
- `TEA_FILTER_DROPPED_REPOS`: GitHub returns repos, TEA backend returns fewer/zero
- `FRONTEND_DROPPED_REPOS`: backend returns repos, UI renders zero
- `WRONG_ENDPOINT_OR_STATE`: frontend/backed request path/state mismatch
- `OTHER`: only with exact evidence

## Repair rules

- If `TOKEN_NO_REPO_ACCESS`: DO NOT change code. Return truthful remediation only.
- If `TEA_FILTER_DROPPED_REPOS`: fix the smallest backend filter/normalization bug. Fine-grained PAT repos must not be dropped only because classic `permissions.push/admin/maintain` is absent.
- If `FRONTEND_DROPPED_REPOS`: fix the smallest frontend mapping/render/state bug.
- If `WRONG_ENDPOINT_OR_STATE`: fix the smallest endpoint/base-path/cache/state mismatch.

Do not hardcode `TEA` and do not fabricate repositories.

## PAT remediation if GitHub itself returns zero repos

Report these exact checks to the user:
- Resource owner must be the account/org that owns `TEA`
- Repository access must include `TEA` or `All repositories`
- `Contents` permission should be `Read and write` for later Push/Sync
- Metadata read access must remain available

Do not ask for or print the token.

## Scope / safety

- Preserve FIX 16 persistent token storage
- Preserve Super Admin gate
- No git commit/push/pull/fetch/merge/rebase/reset/clean
- No Initial Push
- No nginx/systemd/ports/dependency changes
- No token output/logging
- Normal OpenHands UI must remain healthy

## Validation

If no code change is needed, do not restart.
If a code change is required, validate the changed layer before restart:
- backend syntax/import PASS if backend changed
- typecheck/build PASS if frontend changed
- restart only NEW TEA service if required

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_17_REAL_TOKEN_REPO_VISIBILITY_DIAG
TOKEN_PERSISTED=YES|NO
GITHUB_USER_HTTP=
GITHUB_LOGIN=
GITHUB_REPOS_HTTP=
GITHUB_RAW_REPO_COUNT=
GITHUB_REPO_NAMES_SANITIZED=<comma-separated full_name values or NONE>
TEA_RETURNED_BY_GITHUB=YES|NO
TEA_REPO_FULL_NAME=
TEA_PERMISSION_METADATA=
TEA_BACKEND_REPO_COUNT=
UI_REPO_COUNT=
ROOT_CAUSE_CLASS=TOKEN_NO_REPO_ACCESS|TEA_FILTER_DROPPED_REPOS|FRONTEND_DROPPED_REPOS|WRONG_ENDPOINT_OR_STATE|OTHER
ROOT_CAUSE=
CODE_CHANGE_REQUIRED=YES|NO
BACKEND_FIX=PASS|FAIL|NOT_NEEDED
FRONTEND_FIX=PASS|FAIL|NOT_NEEDED
TEA_VISIBLE_AFTER=PASS|FAIL|TOKEN_NO_ACCESS
EMPTY_REPO_STATE=PASS|FAIL|NOT_REACHED
INITIAL_PUSH_EXECUTED=NO
SUPERADMIN_GATE=PASS|FAIL
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=NONE|<paths>
STATUS=READY|NEEDS_PAT_ACCESS_FIX|FAILED
```

`STATUS=NEEDS_PAT_ACCESS_FIX` is correct only if `/user` succeeds and GitHub itself returns zero accessible repositories.
