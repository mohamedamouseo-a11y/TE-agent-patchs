# FIX 07 — TEA 502 runtime recovery

Scope: NEW TEA only at `/opt/TBA` and `tea.tamiyouz.com`.

The public domain currently returns nginx 502 after the latest application restart.

## Required diagnosis

Identify the live request chain actually used by production:

- nginx upstream for `/`
- ingress process and listening port
- frontend/Vite process and listening port
- automation backend process/port
- agent server process/port
- `tba-openhands.service` status and recent startup errors
- `/opt/TBA/start-tba.sh` and invoked startup scripts

Compare configured ports to actual listeners. Check whether a child process exited because of a compile/runtime error, wrong startup command, port conflict, or stale/mismatched upstream.

Apply only the smallest confirmed fix needed to restore the existing TEA stack. Do not change Git state, dependencies, Developer Hub behavior, or unrelated services unless the diagnosis proves that change is required.

## Verification

Production must be verified after recovery:

- nginx configuration valid
- configured nginx upstream is listening
- frontend is listening
- ingress is listening
- automation backend is healthy
- agent server is healthy
- `https://tea.tamiyouz.com/` loads OpenHands
- `https://tea.tamiyouz.com/developer-hub` loads
- FIX 06 GitHub sections remain visible
- normal OpenHands UI works
- the TEA stack comes back successfully after one controlled TEA-only service restart

Do not report READY based only on direct backend requests.

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_07_502_RUNTIME_RECOVERY
NGINX_ROOT_UPSTREAM=
TBA_SERVICE_BEFORE=
INGRESS_BEFORE=
VITE_BEFORE=
ROOT_CAUSE=
FIX_APPLIED=
TBA_SERVICE_AFTER=PASS|FAIL
INGRESS=PASS|FAIL
VITE=PASS|FAIL
AUTOMATION=PASS|FAIL
AGENT_SERVER=PASS|FAIL
NGINX=PASS|FAIL
DOMAIN_HOME=PASS|FAIL
DOMAIN_DEVHUB=PASS|FAIL
FIX06_SECTIONS=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
RESTART_SURVIVAL=PASS|FAIL
CHANGED_FILES=<paths or NONE>
STATUS=READY|FAILED
```