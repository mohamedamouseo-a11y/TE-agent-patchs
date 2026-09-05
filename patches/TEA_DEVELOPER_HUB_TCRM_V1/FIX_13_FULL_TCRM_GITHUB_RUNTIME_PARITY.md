# FIX 13 — Full TCRM GitHub Module Runtime Parity

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Baseline guard

FIX 12 is the ONLY accepted baseline.

Expected live file:
`/opt/TBA/src/components/features/developer-hub/github-panel.tsx`

Expected FIX 12 SHA256:
`9b71665a083d801ce1f9861a9902b2ecc7a209bc3c8fbb4c49710e350bed3548`

Before editing, run REAL `sha256sum` on that file.

- If the real SHA equals the expected FIX 12 SHA, continue.
- If it differs, do NOT overwrite blindly. Inspect the current file, preserve valid newer work, and report the mismatch in `BASELINE_SHA_MATCH=NO`.

## Confirmed production UI problem

The FIX 12 production screenshot proves the page is healthy but the GitHub area is still only the reduced/basic connection UI.

Visible now:
- Repository Status
- GitHub Connection
- Repository/account card
- Branch selector
- Operations
- Audit
- Push Control
- MCP
- AI Context
- Source Export

Missing from the REAL rendered GitHub module:
- Synchronization Status
- Push Mode
- Review Push
- Review Pull
- Review Sync
- Safe Cleanup
- Review & Execute
- changed-files / conflicts / blockers preview
- Commit Message
- Review/Execute Progress
- Operation Log

Therefore `FULL_GITHUB_PANEL_VISIBLE=YES` from FIX 12 was not sufficient visual parity proof.

## Source of truth

Match TCRM Main Developer Hub interaction model as closely as possible.

Repository:
`mohamedamouseo-a11y/TCRM-MAIN-Tamiyouz-CRM-`

Pinned commit:
`78273711727e834ca88029e39ff0f6ae302d427a`

Primary UI reference:
`client/src/components/DeveloperHubTab.tsx`

Primary backend reference:
`server/routes/developerHub.ts`

Also read the existing TEA parity spec already in this patch bundle:
`GITHUB_UI_TCRM_PARITY_V1.md`

Do NOT invent a generic Git UI. Preserve TEA/OpenHands native styling/components.

## Task

1. Inspect the ACTUAL production-served source first:
   - `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`
   - `/opt/TBA/src/routes/developer-hub.tsx`
   - `/opt/TBA/src/hooks/query/use-developer-hub.ts`
   - `/opt/TBA/src/api/developer-hub-service.api.ts`
   - `/opt/TBA/src/types/developer-hub.ts`

2. Determine exactly why only the reduced GitHub UI is rendered despite the previous parity work.

Possible causes must be PROVEN, not assumed:
- FIX 12 repaired syntax by leaving only a simplified component body
- full sections exist but are behind an incorrect conditional
- full component is not mounted
- current hooks/types/API bindings do not expose the full workflow state
- stale/duplicate component or route binding

3. Fix the REAL cause with the smallest safe change.

4. The production GitHub panel must visibly include and functionally wire ALL of the following:

### Connection & repository
- connected account/status
- connect/disconnect
- repository selector / selected repository
- refresh repositories if supported by current API
- branch selector / selected branch
- refresh branches if supported by current API
- save selection if required by current API

### Synchronization Status
- local branch / selected remote repo+branch
- local ahead / remote ahead
- dirty / sync state where available
- last sync/status information where available

### Push Mode
- Off
- Review
- Auto only if the existing TEA backend/API already supports it

IMPORTANT: even if an `auto` mode exists in settings, the coding/execution agent itself must NEVER autonomously trigger Git writes. Human Super Admin controls remain authoritative.

### Review actions
Visible controls for:
- Review Push
- Review Pull
- Review Sync
- Safe Cleanup only if supported by the existing backend

Every mutation remains:
`Review / Preview -> explicit human Execute`

### Review & Execute
After successful review, show the real preview data where provided by the current API:
- action
- repo / branch
- current branch
- ahead/behind
- dirty/sync state
- expected action
- changed files
- conflicts
- blockers
- Push Control gates
- stale-review state

Execute must remain disabled until a valid non-stale successful review exists.

### Commit message
Show the optional commit-message field only for operations where the existing backend supports/requires it.

### Progress and logs
Show:
- review progress
- execute progress
- current step
- operation ID/status where available
- sanitized operation log
- recovered active operation state after reload where current hooks/API support it

5. Do NOT fake parity by adding static headings, placeholder cards, mock PASS values, or disabled shells with no API wiring.

6. Reuse the CURRENT persistent TEA Developer Hub API under:
`/api/automation/v1/developer-hub/*`

Do NOT create another backend/service/port.

7. Preserve all existing Developer Hub cards outside the GitHub module:
- Repository Status
- Operations
- Audit
- Push Control
- MCP
- AI Context
- Source Export

8. Preserve Super Admin protection and current human-only Git policy.

9. Do NOT touch:
- nginx
- systemd
- ports
- React Router/dependencies
- old TEAgent server
- normal OpenHands conversation behavior
- backend unless a specific missing API capability required by the already-defined TEA parity contract is proven absent

If a backend or another file change is truly required, make only the minimum existing-architecture change and list every changed file. No new services.

10. The executor applying this patch must NOT perform Git write/network operations itself:
- no git commit
- no git push
- no git pull
- no git fetch
- no merge/rebase/reset/clean

The GitHub controls being implemented are for the HUMAN Super Admin inside Developer Hub only.

## Validation gate BEFORE restart

Before any TEA service restart:

1. Run real parse/TypeScript validation.
2. Run the repository's current build validation.
3. Capture real `sha256sum` after the edit.
4. SHA must actually change if `github-panel.tsx` changed.

If parse/type/build fails, do NOT restart. Fix only errors caused by this patch.

## Production browser verification

After validation passes, restart ONLY the NEW TEA service if required.

Open a REAL browser at:
`https://tea.tamiyouz.com/developer-hub`

Do not accept source grep, curl 200, API 200, or DOM text assertions alone as visual proof.

Verify the actual rendered UI visibly contains:
1. Connection & Repository
2. Synchronization Status
3. Push Mode
4. Review Push
5. Review Pull
6. Review Sync
7. Safe Cleanup when supported
8. Review & Execute
9. Commit Message when applicable
10. Progress
11. Operation Log

Also verify normal OpenHands UI still works.

Capture TWO real screenshots:
- upper GitHub module area
- lower Review/Execute/Progress/Operation Log area

A report cannot be READY if the screenshots do not visibly prove the required sections.

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_13_FULL_TCRM_GITHUB_RUNTIME_PARITY
FILE=/opt/TBA/src/components/features/developer-hub/github-panel.tsx
REAL_SHA256_BEFORE=
EXPECTED_FIX12_SHA=9b71665a083d801ce1f9861a9902b2ecc7a209bc3c8fbb4c49710e350bed3548
BASELINE_SHA_MATCH=YES|NO
REAL_SHA256_AFTER=
SHA_CHANGED=YES|NO
ROOT_CAUSE=
PARSE_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL
BUILD_VALIDATION=PASS|FAIL
CONNECTION_REPOSITORY_VISIBLE=PASS|FAIL
SYNC_STATUS_VISIBLE=PASS|FAIL
PUSH_MODE_VISIBLE=PASS|FAIL
REVIEW_PUSH_VISIBLE=PASS|FAIL
REVIEW_PULL_VISIBLE=PASS|FAIL
REVIEW_SYNC_VISIBLE=PASS|FAIL
SAFE_CLEANUP_VISIBLE=PASS|FAIL|NOT_SUPPORTED
REVIEW_EXECUTE_VISIBLE=PASS|FAIL
PREVIEW_DETAILS_VISIBLE=PASS|FAIL
COMMIT_MESSAGE_VISIBLE=PASS|FAIL|NOT_APPLICABLE
PROGRESS_VISIBLE=PASS|FAIL
OPERATION_LOG_VISIBLE=PASS|FAIL
API_WIRING_REAL=PASS|FAIL
STALE_REVIEW_PROTECTION=PASS|FAIL
PUSH_CONTROL_WIRED=PASS|FAIL
SUPERADMIN_GATE=PASS|FAIL
DOMAIN_DEVHUB=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
SCREENSHOT_UPPER_PATH=
SCREENSHOT_LOWER_PATH=
CHANGED_FILES=
STATUS=READY|FAILED
```

`STATUS=READY` is allowed only if parse/type/build pass, the production browser visibly renders the full TCRM-style GitHub workflow, the controls are backed by real current TEA API/state (not static placeholders), both screenshots are captured, and normal OpenHands remains healthy.