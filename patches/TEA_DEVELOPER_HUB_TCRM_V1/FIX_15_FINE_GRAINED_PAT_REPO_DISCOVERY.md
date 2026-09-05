# FIX 15 — Fine-grained PAT repository discovery repair

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed production symptom

After FIX 14, the human Super Admin enters a GitHub fine-grained personal access token and presses Connect. Authentication appears accepted, but the real production UI shows:

- Repository: `No repositories`
- Branch: `No branches`
- Save Selection disabled
- local repository status still loads

The intended real GitHub repository is named `TEA` and is currently empty.

Therefore FIX 14 is NOT accepted as complete despite its report claiming `TEA_REPO_VISIBLE=PASS`.

## Important rule

Do NOT assume the token itself is wrong and do NOT assume the backend filter is wrong. Prove which case applies from the live GitHub API response without printing or logging the token.

## Task

### 1. Inspect the actual live implementation

Inspect only the NEW TEA source/state needed for this issue:
- `/opt/TBA/openhands/automation/developer_hub/router.py`
- persistent Developer Hub state/config used for the GitHub token
- `/opt/TBA/src/api/developer-hub-service.api.ts`
- `/opt/TBA/src/hooks/query/use-developer-hub.ts`
- `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`

Do not change anything before collecting the live evidence below.

### 2. Prove token authentication and repository visibility separately

Using the stored token internally on the server, call GitHub REST API read-only and record ONLY sanitized metadata:

A. `GET /user`
- HTTP status
- authenticated login

B. `GET /user/repos?per_page=100&page=1&sort=updated&affiliation=owner,collaborator,organization_member`
- HTTP status
- raw repository count BEFORE any TEA-side filtering
- for each result, record only `full_name`, `private`, `default_branch`, and the permission object/role if returned

NEVER print the Authorization header or token.

If pagination is needed, continue until the result page is shorter than 100.

### 3. Distinguish the real failure mode

Classify exactly one primary cause:

- `TOKEN_NO_REPO_ACCESS` = token authenticates but GitHub returns zero accessible repositories
- `TEA_FILTER_DROPPED_REPOS` = GitHub returns repositories but TEA filters all/some of them incorrectly
- `FRONTEND_DROPPED_REPOS` = backend returns repositories but UI renders `No repositories`
- `STATE_OR_ENDPOINT_MISMATCH` = UI calls wrong endpoint/base path or reads stale response
- another proven cause with exact evidence

### 4. Fine-grained PAT compatibility

The repository discovery UI must list repositories that GitHub actually returns to the token even when the GitHub response does NOT contain the classic `permissions.push/admin/maintain` shape.

Do NOT discard a repository merely because `repo.permissions` is missing or has a different shape.

Normalize permission safely:
- use `permissions` when present
- use `role_name` / equivalent returned metadata when present
- otherwise mark permission as `unknown`

Repository discovery is a READ operation. It must not require proven push permission just to show the repository.

Write capability must instead be enforced at Review/Execute time.

### 5. Empty `TEA` repository support

If GitHub returns the real repository named `TEA`, it must appear in the dropdown even if it is empty and has no branches.

After selection:
- repository selection succeeds
- branch UI shows explicit empty-remote state instead of generic failure
- suggest/allow initial target branch `main` if GitHub has no branch yet
- `Review Push` can produce an initial-push review
- actual push remains human-only and MUST NOT be executed by this patch task

### 6. If GitHub genuinely returns ZERO repositories

Do NOT fabricate a repository and do NOT hardcode `TEA`.

Return a truthful UI/API state such as:
`GitHub token authenticated, but no repositories are accessible to this token.`

For a fine-grained PAT, show a concise remediation hint in the UI without exposing secrets:
- ensure the token Resource owner is the account/org that owns `TEA`
- Repository access includes `TEA` (or All repositories)
- repository Contents permission is Read and write for later Push/Sync operations
- Metadata read access remains available

The user must be able to reconnect/refresh after correcting PAT access.

### 7. Do not weaken write security

Do NOT allow Push/Pull/Sync Execute just because a repository is visible.

Before any write operation, verify the token can perform the required action using the existing safe GitHub/Git review logic. If permission is insufficient, block Execute with a clear reason.

### 8. Frontend behavior

The production UI must distinguish:
- token invalid
- token valid but zero accessible repos
- repositories loading
- repository API error
- repositories available
- selected repository empty/no branches

Do not show `No repositories` as the only state for all failures.

### 9. Scope protection

Preserve FIX 13 full GitHub module and FIX 14 real repository/empty-remote work.

Do NOT modify:
- nginx
- systemd
- ports
- React Router/dependencies
- old TEAgent server
- normal OpenHands conversation behavior
- unrelated Developer Hub cards

No new service or port.

Executor restrictions:
- no git commit/push/pull/fetch/merge/rebase/reset/clean against the TEA source repo
- no actual initial push
- GitHub REST read-only diagnostics are allowed
- never print/log/store the raw token outside the existing secure state mechanism

## Validation

Before restart:
- backend syntax/import validation PASS
- frontend parse/type validation PASS if frontend changed
- build validation PASS if frontend changed

After restart only if required:

1. Reconnect/refresh using the already stored token without exposing it.
2. Compare sanitized GitHub raw repo count vs backend returned repo count vs UI rendered repo count.
3. If GitHub returns `TEA`, verify it is visible/selectable in production.
4. If GitHub returns zero repositories, verify production displays the explicit token-access remediation message instead of pretending discovery succeeded.
5. Normal OpenHands UI must remain healthy.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_15_FINE_GRAINED_PAT_REPO_DISCOVERY
GITHUB_USER_HTTP=
GITHUB_LOGIN=
GITHUB_REPOS_HTTP=
GITHUB_RAW_REPO_COUNT=
TEA_BACKEND_REPO_COUNT=
UI_REPO_COUNT=
ROOT_CAUSE_CLASS=
ROOT_CAUSE=
TOKEN_AUTH_VALID=PASS|FAIL
TOKEN_HAS_ANY_REPO_ACCESS=YES|NO
TEA_RETURNED_BY_GITHUB=YES|NO
TEA_PERMISSION_METADATA=
TEA_FILTER_REPAIRED=PASS|FAIL|NOT_NEEDED
FRONTEND_REPO_RENDER=PASS|FAIL|NOT_NEEDED
ZERO_REPO_EXPLICIT_UI=PASS|FAIL|NOT_APPLICABLE
TEA_REPO_VISIBLE=PASS|FAIL|TOKEN_NO_ACCESS
TEA_REPO_SELECTABLE=PASS|FAIL|TOKEN_NO_ACCESS
EMPTY_REPO_STATE=PASS|FAIL|NOT_REACHED
INITIAL_PUSH_REVIEW=PASS|FAIL|NOT_REACHED
INITIAL_PUSH_EXECUTED=NO
WRITE_PERMISSION_STILL_GATED=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
BACKEND_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL|NOT_CHANGED
BUILD_VALIDATION=PASS|FAIL|NOT_CHANGED
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=
STATUS=READY|NEEDS_TOKEN_PERMISSION_FIX|FAILED
```

`STATUS=READY` only if the real GitHub repository results are represented truthfully in production.

If the token authenticates but GitHub returns zero repos, return `STATUS=NEEDS_TOKEN_PERMISSION_FIX`; that is not a code failure and must not be hidden with fake data.