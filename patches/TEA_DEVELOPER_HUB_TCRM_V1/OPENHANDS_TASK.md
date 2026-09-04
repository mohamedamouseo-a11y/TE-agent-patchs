# Apply task — TEA Developer Hub TCRM V1

Work only on the NEW TEA server and install:

- server: `158.220.119.80`
- target: `/opt/TBA`
- domain: `https://tea.tamiyouz.com`

Do not modify the old TEAgent/OpenHands instance.

## Task

1. Read `README.md`, `REFERENCE.md`, `VERIFY.md` and `manifest.json` in this patch bundle.
2. Use the pinned TCRM reference commit from `REFERENCE.md`. Inspect only the reference Developer Hub files/helpers needed to preserve its current behavior.
3. Inspect the minimum TEA integration points in `/opt/TBA`:
   - existing admin/settings navigation and routes
   - existing auth/role model and server-side authorization hooks
   - existing frontend API/service pattern
   - existing backend/API extension pattern
   - current config/secret persistence pattern
   - current UI primitives/theme/RTL behavior
4. Implement the Developer Hub inside the EXISTING TEA/OpenHands architecture. Do not add a second web framework, auth system or standalone service.
5. Match the pinned TCRM Developer Hub behavior and structure described in `REFERENCE.md`: Status, GitHub connection, repository/branch selection, Review → Execute Push/Pull/Sync, Git, Push Control, MCP, AI Context, Source Code export, operation recovery/progress, preflight/security and audit.
6. Enforce Super Admin authorization server-side on every Developer Hub backend capability. Hiding a menu item is not authorization.
7. Git mutation policy:
   - human Super Admin action only
   - no agent-triggered commit/push/pull/sync
   - no force push
   - no destructive reset/clean
   - no arbitrary shell UI
   - trusted TEA repo root only
   - secrets server-side and redacted
   - preview/review state bound to the exact repository/branch/HEAD/fingerprint before execute
   - serialize write operations and keep sanitized audit/recovery state
8. Developer Hub must NOT restart/deploy TEA. It may report that an external restart/deploy is required.
9. Do not change OpenHands dependencies, ports, nginx, systemd, domain config or unrelated functionality.
10. Run only the existing relevant checks/build needed for the touched code. Do not fix unrelated failures.
11. Execute `VERIFY.md` acceptance checks.

## Target-runtime rule

The external executor applying this patch may restart **only the NEW TEA service** after a successful build if loading the implemented code genuinely requires it. Developer Hub code itself must never expose or autonomously trigger restart/deploy.

## Completion report

Return only:

```text
PATCH=TEA_DEVELOPER_HUB_TCRM_V1
TARGET=/opt/TBA
SUPERADMIN_BACKEND_GUARD=PASS|FAIL
TCRM_PARITY=PASS|FAIL
GITHUB_MANUAL_ONLY=PASS|FAIL
PREVIEW_EXECUTE=PASS|FAIL
PUSH_CONTROL=PASS|FAIL
MCP=PASS|FAIL
AI_CONTEXT=PASS|FAIL
SOURCE_EXPORT=PASS|FAIL
OPERATION_RECOVERY=PASS|FAIL
AUDIT=PASS|FAIL
NO_AUTO_RESTART_DEPLOY=PASS|FAIL
BUILD_TESTS=<summary>
CHANGED_FILES=<comma-separated paths>
TEA_RESTARTED_BY_EXECUTOR=YES|NO
DOMAIN_SMOKE_TEST=PASS|FAIL
STATUS=READY_FOR_REVIEW|FAILED
```