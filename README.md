# TE Agent Patches

Patch bundles for TEA / TEAgent installations.

## Active patch

### `patches/TEA_DEVELOPER_HUB_TCRM_V1/`

Developer Hub for the **new TEA OpenHands instance** at `tea.tamiyouz.com`, ported from the current TCRM Developer Hub behavior pinned at commit `78273711727e834ca88029e39ff0f6ae302d427a`.

Apply entrypoint:

`patches/TEA_DEVELOPER_HUB_TCRM_V1/OPENHANDS_TASK.md`

Verification:

`patches/TEA_DEVELOPER_HUB_TCRM_V1/VERIFY.md`

Key policy:

- Super Admin only, server-enforced
- TCRM-style Review → Execute Git workflow
- Git writes are manual human Super Admin actions only
- no autonomous agent push/pull/commit/sync
- no force push or destructive reset/clean
- credentials stay server-side
- Developer Hub never restarts or deploys TEA

## Legacy

`phase-00-developer-hub/` is the earlier TEAgent self-development specification. It is retained for history only and is **not** the active implementation target for the new TEA instance.
