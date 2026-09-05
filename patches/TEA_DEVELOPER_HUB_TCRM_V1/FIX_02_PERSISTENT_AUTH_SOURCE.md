# FIX 02 — Persistent Developer Hub Auth/Config Source

Apply only to the NEW TEA instance on `158.220.119.80` at `/opt/TBA`.

Do not perform Git writes. Do not touch the old TEAgent server.

## Confirmed issue

FIX_01 passed behavior checks, but its changed-files report shows Developer Hub auth/config/router edits inside multiple `/root/.cache/uv/archive-v0/.../site-packages/openhands/automation/` paths.

Custom production behavior must not depend on uv cache/package archive files because those paths can change or disappear after dependency refresh/reinstall/rebuild.

`/opt/TBA/openhands/automation/developer_hub/router.py` is already persistent, but the Super Admin/auth/config changes must also resolve from persistent `/opt/TBA` source.

## Task

1. Inspect the exact diffs currently present in the uv-cache copies of:
   - `openhands/automation/auth.py`
   - `openhands/automation/config.py`
   - `openhands/automation/developer_hub/router.py`

2. Move ONLY the required Developer Hub-specific auth/config changes into persistent OpenHands source under `/opt/TBA/openhands/automation/`, preserving existing upstream behavior.

3. Ensure startup imports the persistent `/opt/TBA/openhands` source first. Do not add another service, port, framework, package or dependency.

4. Do not rely on modifying `/root/.cache/uv`, `site-packages`, or archive directories. After persistent source is correct, cache copies must not be required for Developer Hub behavior.

5. Verify the running automation backend imports these modules from persistent source using runtime introspection, for example the actual module `__file__` paths. Required runtime paths must resolve under `/opt/TBA/openhands/...` for Developer Hub custom auth/config/router code.

6. Restart only NEW TEA services if required and verify restart survival.

7. Re-run the critical Developer Hub checks through `https://tea.tamiyouz.com`:
   - authenticated Super Admin allowed
   - authenticated non-Super-Admin receives 403
   - Developer Hub API works through production ingress
   - Developer Hub UI/navigation works
   - normal OpenHands home/conversation UI still works

## Constraints

- No Git writes.
- No nginx/port changes unless the current known-good 18003 route is accidentally broken; otherwise leave networking untouched.
- No unrelated cleanup or refactor.
- Do not delete uv cache just to prove the result.
- Do not report PASS based only on file existence; prove the live process imports persistent source.

Return only:

```text
FIX=TEA_DEVELOPER_HUB_TCRM_V1_FIX_02
PERSISTENT_AUTH_SOURCE=PASS|FAIL
PERSISTENT_CONFIG_SOURCE=PASS|FAIL
PERSISTENT_ROUTER_SOURCE=PASS|FAIL
AUTH_RUNTIME_FILE=<path>
CONFIG_RUNTIME_FILE=<path>
ROUTER_RUNTIME_FILE=<path>
UV_CACHE_REQUIRED=YES|NO
SUPERADMIN_API=PASS|FAIL
NON_SUPERADMIN_403=PASS|FAIL
DOMAIN_DEVHUB_UI=PASS|FAIL
NORMAL_OPENHANDS_UI=PASS|FAIL
RESTART_SURVIVAL=PASS|FAIL
CHANGED_FILES=<paths>
STATUS=READY|FAILED
```