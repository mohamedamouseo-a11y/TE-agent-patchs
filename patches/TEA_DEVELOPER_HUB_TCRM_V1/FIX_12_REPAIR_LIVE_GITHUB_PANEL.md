# FIX 12 — Repair the actual live GitHub panel

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Confirmed state

Independent verification proved FIX 11 did **not** persist:

- live file: `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`
- real SHA256 is still `f26e22c941542445e33a73bf9798235e0ca20f23749669ab51bbab655dbaccda`
- parse validation: FAIL
- typecheck: FAIL
- Vite overlay: still present
- Developer Hub production route: FAIL

Therefore previous READY reports are invalid.

## Task

1. Work on the actual file only:
   `/opt/TBA/src/components/features/developer-hub/github-panel.tsx`

2. Before editing, capture REAL shell output from:
   - `sha256sum /opt/TBA/src/components/features/developer-hub/github-panel.tsx`
   - `nl -ba /opt/TBA/src/components/features/developer-hub/github-panel.tsx | sed -n '1,80p'`

3. Fix the malformed TSX structure in the real file. Start with the smallest correction around the component declaration / opening return structure. Do not claim success from source inspection alone.

4. If the first syntax repair exposes additional parse/type errors, fix only those errors in this same file until the file is syntactically valid and typechecks. Do not rewrite unrelated modules.

5. Preserve the full Developer Hub GitHub module functionality already required by this bundle:
   - Connection & Repository
   - Repository selector
   - Branch selector
   - Push Mode
   - Review Push / Pull / Sync
   - Safe Cleanup
   - Review & Execute
   - Commit message
   - Progress
   - Operation Log

6. Do NOT touch backend, nginx, ports, systemd, dependencies, auth, or unrelated frontend files unless the current file cannot compile without an already-existing import path correction. If another file must change, STOP and report `NEEDS_EXTRA_FILE=<path>` instead of editing it.

7. Before any restart, run the repository's current frontend parse/type/build validation. A restart is forbidden while parse/type validation fails.

8. After validation passes, capture REAL shell output again:
   - `sha256sum /opt/TBA/src/components/features/developer-hub/github-panel.tsx`
   - first 80 numbered lines

9. Confirm `SHA256_AFTER` is different from the confirmed broken SHA above. Do not fabricate or manually type a hash.

10. Restart only the NEW TEA service if needed to reload the fixed frontend.

11. Verify in the real production browser:
   - no Vite error overlay
   - `/developer-hub` renders
   - full GitHub module is visible
   - normal OpenHands home UI still works

12. Capture one real production screenshot after the fix.

## Prohibited

- no Git commit/push/pull/fetch/merge/rebase
- no edits to old TEAgent server
- no placeholder hashes
- no READY based only on grep/source text
- no restart before validation passes

## Completion report

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_12_REPAIR_LIVE_GITHUB_PANEL
FILE=/opt/TBA/src/components/features/developer-hub/github-panel.tsx
REAL_SHA256_BEFORE=
REAL_SHA256_AFTER=
SHA_CHANGED=YES|NO
ROOT_CAUSE=
PARSE_VALIDATION=PASS|FAIL
TYPECHECK=PASS|FAIL
BUILD_VALIDATION=PASS|FAIL
VITE_OVERLAY_GONE=YES|NO
DOMAIN_DEVHUB=PASS|FAIL
FULL_GITHUB_PANEL_VISIBLE=YES|NO
NORMAL_OPENHANDS_UI=PASS|FAIL
SCREENSHOT_PATH=
CHANGED_FILES=
NEEDS_EXTRA_FILE=NONE|<path>
STATUS=READY|FAILED
```

STATUS=READY is allowed only if the real hash changed, validation passes before restart, production browser works, and the full GitHub panel is visibly rendered.