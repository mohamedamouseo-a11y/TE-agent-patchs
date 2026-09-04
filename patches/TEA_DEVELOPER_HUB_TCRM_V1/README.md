# TEA Developer Hub — TCRM V1

Application-ready port bundle for the **new TEA OpenHands instance**.

## Target

- Product: TEA (Tamiyouz Execution Agent)
- Server: `158.220.119.80`
- Current install path: `/opt/TBA`
- Domain: `https://tea.tamiyouz.com`
- Base application: clean `OpenHands/OpenHands` checkout

## Exact reference

Port the current TCRM Developer Hub behavior and UX from:

- Repository: `mohamedamouseo-a11y/TCRM-MAIN-Tamiyouz-CRM-`
- Pinned reference commit: `78273711727e834ca88029e39ff0f6ae302d427a`
- Primary UI: `client/src/components/DeveloperHubTab.tsx`
- Primary backend: `server/routes/developerHub.ts`

The executor must use this pinned revision as the behavior/reference source. Adapt framework integration only where TEA/OpenHands differs.

## Goal

Create a **Super Admin-only Developer Hub inside TEA** with the same operational model as TCRM:

- repository/branch status and GitHub connection
- manual repository + branch selection
- review/preview before Push / Pull / Sync
- Push Control / preflight status
- Git operations panel
- MCP controls
- AI Context controls
- Source Code export
- operation progress, recovery and audit
- security/preflight status
- bilingual/RTL-compatible UI where the TEA shell supports it

## Non-negotiable operating model

The coding agent is **not allowed to autonomously publish or operate Git**. GitHub connection and all Git write actions are human Super Admin controls inside Developer Hub.

Developer Hub itself must never auto-restart, auto-deploy, or replace the running TEA instance.

Credentials remain server-side and must never be rendered, logged, returned in API payloads, or committed.

## Apply

Read `OPENHANDS_TASK.md` and execute it against `/opt/TBA` on the target server. Then run `VERIFY.md`.

This bundle is a port specification/reference pack, not a blind unified diff: TCRM and OpenHands use different server/UI architecture, so the executor must bind the reference behavior to the existing TEA architecture instead of introducing TCRM's framework into OpenHands.