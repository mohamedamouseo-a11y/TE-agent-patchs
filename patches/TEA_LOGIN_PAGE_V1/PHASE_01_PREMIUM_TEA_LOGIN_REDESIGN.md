# TEA LOGIN PAGE V1 — Phase 01 Premium TEA Login Redesign

Target: NEW TEA only (`/opt/TBA`, `https://tea.tamiyouz.com`).

## Goal

Redesign the TEA authentication/login experience so it feels native to **Tamiyouz Execution Agent (TEA)**, not like a generic SaaS login page.

The design direction is a premium dark execution/AI workspace aesthetic:

- deep near-black background
- warm champagne / muted gold accents
- subtle glass + border layers
- restrained glow, no flashy neon
- clear enterprise typography
- visual language centered on agents, automation, repositories, execution, visibility, and human-controlled AI

This is a UI/UX redesign only. Preserve the existing authentication behavior and backend contract exactly unless a frontend compatibility fix is strictly required.

---

## Visual concept

### Overall composition

Desktop layout: 2-column split.

- **Left panel (approx. 36–40%)**: login form
- **Right panel (approx. 60–64%)**: TEA brand/execution visual world

Full viewport height. No ordinary centered white login card.

### Brand system

Use TEA branding prominently:

`TEA`

Subtitle:

`TAMIYOUZ EXECUTION AGENT`

Suggested supporting line near logo:

`Better ideas. Better execution. A more capable you.`

Color direction:

- background: #0B0D0F / #0E1013 range
- panel surfaces: #111419 / #15181D range
- primary text: warm off-white
- secondary text: muted cool gray
- accent: champagne / muted gold / bronze, not saturated yellow
- success: restrained green only for status indicators

Do not introduce bright purple/blue gradients.

---

## Left login panel

### Header

Large heading:

`Welcome to TEA`

Subheading:

`Sign in to your Tamiyouz Execution Agent workspace.`

### Form

Preserve the real existing auth fields and submission flow.

Expected presentation:

- Email field
- Password field
- password visibility toggle if the current auth implementation supports it safely
- Keep me signed in only if current auth behavior already supports persistence
- Forgot password only if a real route/action already exists
- Primary CTA: `Sign in to TEA`

Inputs:

- dark transparent surface
- 1px neutral border
- subtle gold focus ring
- 12–14px radius
- 48–54px desktop height
- clear accessible labels above fields
- error state must remain readable and accessible

### SSO

Do **not** add fake Google or Microsoft auth.

If the current TEA authentication layer already exposes working Google/Microsoft/OIDC actions, render them in a secondary section.

If no real SSO flow exists, omit the SSO buttons entirely.

Never create visual-only login actions that do nothing.

### Footer

Keep minimal:

- Privacy / Terms / Help only if real destinations exist
- otherwise omit dead links

---

## Right TEA brand world

This panel should visually explain what TEA is before login.

### Primary headline

Eyebrow:

`IDEAS → AGENTS → EXECUTION → REAL IMPACT`

Headline:

`Execute Faster with Intelligent Agents`

Supporting copy:

`Automate workflows, manage agent contexts, and operate with confidence from one unified workspace.`

### Hero visualization

Build a premium **TEA execution workspace composition**, not a stock photo.

Preferred implementation: native HTML/CSS/SVG using the project's existing icon system and design primitives so it remains lightweight and sharp.

Visual should suggest:

- Agent Workspace
- repository monitoring
- Plan with Agent
- Execute
- Tests
- Build
- Security
- Deploy / operation complete
- small agent fleet status panel
- subtle node/flow connectors

Example visual hierarchy:

`Monitor Repository → Plan with Agent → Execute`

Then secondary nodes:

`Run Tests`
`Update PR`
`Deploy`

Do not hard-code fake live production data. Use clearly decorative/mock labels only inside the brand illustration.

### Feature rail at bottom

Five concise TEA pillars:

1. `Agents` — Specialized AI agents for real work
2. `Automation` — Turn ideas into actionable workflows
3. `Developer Hub` — Integrate with your tools and code
4. `Execution` — Reliable, observable agent runs
5. `Visibility` — Track progress, results, and impact

Use existing icons where possible.

---

## Motion

Motion should be subtle and premium.

Allowed:

- very slow ambient gold glow
- small status indicator pulse
- gentle flow-line animation
- 150–250ms UI transitions

Avoid:

- large particles
- constant parallax
- aggressive floating cards
- distracting continuous movement around the login form

Respect `prefers-reduced-motion`.

---

## Responsive behavior

### Desktop >= 1200px

Full split layout as described.

### Tablet 768–1199px

Keep the login panel dominant; compress the visual panel and reduce decorative cards.

### Mobile < 768px

- single-column login-first layout
- brand visual becomes a compact atmospheric header/background section
- hide nonessential decorative execution cards
- no horizontal scroll
- CTA full width
- keyboard-safe form spacing

---

## Accessibility

Must preserve/improve:

- semantic labels
- keyboard navigation
- visible focus state
- minimum WCAG AA contrast for functional text
- password toggle accessible name
- error messages linked to fields
- loading/disabled state on submit
- no authentication feedback conveyed by color only

---

## Critical auth-preservation rules

Before editing, inspect the live TEA authentication implementation and identify:

- actual login route
- actual login component(s)
- existing auth API/service calls
- validation
- session/cookie/token behavior
- redirect-after-login behavior
- forgot-password / SSO availability

Then redesign around those real behaviors.

Do NOT:

- replace the auth flow with mock logic
- change credential storage behavior
- move secrets client-side
- add fake auth providers
- expose auth tokens
- break server-side Developer Hub authorization from FIX30+
- change protected-route rules

Successful login must behave exactly as before, only with the new UI.

---

## Production architecture constraints

Preserve the current production architecture:

- production frontend remains built/static/non-HMR
- do not restore `react-router dev` or Vite HMR in production
- `/api/automation` must remain functional
- Developer Hub must remain functional and server-side gated
- do not alter FIX20–FIX32 Developer Hub Git safety behavior

---

## Implementation workflow

1. Inspect current login/auth source first.
2. Record exact files that own the login UI.
3. Make the smallest coherent redesign in those files plus dedicated styling/assets if needed.
4. Prefer reusable local components over one giant route file.
5. Use existing project icon libraries; do not add a heavy UI dependency just for this screen.
6. Avoid remote runtime images/CDN dependencies for the core visual.
7. Run parse/type/build validation.
8. Build production frontend.
9. Deploy the built artifact using the current non-HMR production flow.
10. Test real login behavior with an authorized existing test account only if available; never print credentials.
11. Verify redirect after successful login.
12. Verify invalid credentials/error state.
13. Verify `/developer-hub` remains healthy after login for Super Admin.
14. Verify normal OpenHands/TEA routes remain healthy.

---

## Strict restrictions while applying this patch

Do NOT:

- `git push`
- `git pull`
- `git fetch`
- create a remote commit or branch
- call Developer Hub Execute Reviewed Action
- print any token/password/secret
- change backend authentication semantics
- remove server-side authorization gates
- return production to Vite/HMR

Local source, style, asset, build, and service changes required for the login redesign are allowed.

---

## Acceptance criteria

The patch is READY only if all are true:

- login page clearly looks like TEA, not generic OpenHands/SaaS
- real auth flow preserved
- no fake SSO/actions
- desktop/tablet/mobile responsive
- production build passes
- production remains non-HMR
- successful login redirect works
- invalid login feedback works
- `/api/automation` remains healthy
- Developer Hub remains healthy and server-gated
- no secrets exposed in the browser bundle
- no remote Git write executed by the patch

---

## Completion report

Return only:

```text
PATCH=TEA_LOGIN_PAGE_V1_PHASE_01_PREMIUM_TEA_LOGIN_REDESIGN
LOGIN_ROUTE=
LOGIN_SOURCE_FILES_BEFORE=
AUTH_FLOW_PRESERVED=PASS|FAIL
REAL_SSO_AVAILABLE=YES|NO
FAKE_SSO_ADDED=NO
TEA_BRANDING=PASS|FAIL
DESKTOP_LAYOUT=PASS|FAIL
TABLET_LAYOUT=PASS|FAIL
MOBILE_LAYOUT=PASS|FAIL
ACCESSIBILITY=PASS|FAIL
LOGIN_SUCCESS_FLOW=PASS|FAIL|NOT_TESTED_NO_TEST_ACCOUNT
LOGIN_ERROR_FLOW=PASS|FAIL
REDIRECT_AFTER_LOGIN=PASS|FAIL|NOT_TESTED_NO_TEST_ACCOUNT
PRODUCTION_BUILD=PASS|FAIL
PRODUCTION_REBUILD=PASS|FAIL
VITE_HMR_PRESENT=YES|NO
API_AUTOMATION=PASS|FAIL
DEVELOPER_HUB=PASS|FAIL
SERVER_SIDE_AUTH_PRESERVED=PASS|FAIL
CLIENT_SECRET_EXPOSURE=YES|NO
HOME_UI=PASS|FAIL
NORMAL_TEA_UI=PASS|FAIL
REMOTE_WRITE_EXECUTED=NO
CHANGED_FILES=
STATUS=READY|FAILED
```
