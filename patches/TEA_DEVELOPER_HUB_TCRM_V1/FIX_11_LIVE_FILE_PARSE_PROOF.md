# FIX 11 — Live GitHub Panel Parse Proof

Target: NEW TEA only on `158.220.119.80`, path `/opt/TBA`.

## Confirmed failure

The production browser still shows the same Vite parse overlay for:

`/opt/TBA/src/components/features/developer-hub/github-panel.tsx`

with:

`[PARSE_ERROR] Unexpected token`

around line 16 near:

`const { t } = useTranslation("openhands");`

Therefore previous FIX 10 READY report is invalid. Do not assume the parse error was fixed.

## Task

1. Do not restart anything yet.
2. Inspect the actual live file currently on disk:
   - print lines 1-40 with line numbers
   - record SHA256 of `github-panel.tsx`
3. Identify the exact syntax error ABOVE or AROUND line 16. The error location may be caused by malformed syntax on the preceding line(s).
4. Apply the smallest valid TSX correction only to:
   `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`
5. Immediately re-print lines 1-40 and record the new SHA256.
6. Run a real parser/type validation against the changed file/project BEFORE restart. Do not report PASS from visual source inspection.
7. If validation fails, stop and fix the actual syntax error before restart.
8. Only after parser/type validation passes, restart only the NEW TEA service if required.
9. Verify the public browser no longer shows the Vite overlay.
10. Verify `/developer-hub` renders and the full GitHub panel labels are visible.
11. Do not rewrite the full component unless absolutely necessary. Preserve the current full GitHub module implementation.
12. No Git writes. No nginx/port/backend changes.

## Mandatory proof

Return the exact before/after first 25 lines of the file in the report, plus both SHA256 values.

`STATUS=READY` is forbidden unless:
- parser/type validation passes after the edit
- Vite overlay is gone in the production browser
- the public Developer Hub page renders

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_11_LIVE_FILE_PARSE_PROOF
FILE=/opt/TBA/src/components/features/developer-hub/github-panel.tsx
SHA256_BEFORE=
SHA256_AFTER=
ROOT_CAUSE=
LINES_1_25_BEFORE=<single-line escaped summary>
LINES_1_25_AFTER=<single-line escaped summary>
PARSE_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL
VITE_OVERLAY_GONE=PASS|FAIL
DOMAIN_DEVHUB_UI=PASS|FAIL
FULL_GITHUB_PANEL_VISIBLE=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
CHANGED_FILES=
STATUS=READY|FAILED
```