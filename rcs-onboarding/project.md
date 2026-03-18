# RCS Onboarding Project

## What We're Building

A multi-tenant RCS agent onboarding experience within Burst/Kudosity. Customers create RCS agents (brand profiles), add test devices, send test messages, and submit for Google verification. The backend uses **Google RBM** (RCS Business Messaging) API. The burst monorepo is at `/Users/tbc/repo/burst`.

---

## Current Status

The frontend UI is implemented (local state only, no API integration yet):

- **Agent List page** — table view with mock data, create button, status badges
- **Agent Create Wizard** — 3-step flow (redesigned from original 5-step design, new designs in `v0-designs-NEW/`):
  - Step 1: Sender Details (name, brand, region, billing category, use case)
  - Step 2: Public Profile (display name, description, color, contacts, images, URLs + phone preview)
  - Step 3: Test & Preview (test devices table, add device, send test message)
- **RCS sidebar navigation** — under Channels → RCS (no feature flag gate for now)
- **Feature flag** — `show-rcs-onboarding` in `featureFlagsConfig.ts` (default `true` for dev)

### Branch & PR Status

| Branch | PR | Status | Description |
|---|---|---|---|
| `feat/rcs-onboarding-shell` | [#9155](https://github.com/burstsms/burst/pull/9155) | Open | Feature flag + empty shell |
| `feat/rcs-onboarding-shell` | PR 2 (not yet opened) | In Progress | Agent List + Create wizard + sidebar nav |

---

## Next Steps

### 1. Backend API Integration (highest priority)

The frontend is currently using local state only. Per [backend-integration-plan.md](docs/backend-integration-plan.md):

- [ ] Extend the Go backend (`backend/rcs/core/google/`) with agent CRUD endpoints using Google's Business Communications API
- [ ] Wire up the wizard's "Save and Complete" to call `POST /v2/private/rcs/agents` to create an agent
- [ ] Wire up the agent list page to call `GET /v2/private/rcs/agents` to fetch existing agents
- [ ] Implement test device management APIs (`POST/DELETE /v2/private/rcs/agents/{id}/testers`)
- [ ] Implement send test message API (`POST /v2/private/rcs/agents/{id}/test-message`)

### 2. Form Validation

- [ ] Add client-side validation (required fields, URL format, hex color format, phone number format)
- [ ] Add character limit enforcement with error states
- [ ] Validate at least one contact method is provided in Step 2

### 3. Image Upload Implementation

- [ ] Banner and logo upload handlers (currently just "Browse" button placeholder)
- [ ] File type/size validation (banner: 1440×448px, 200kB max; logo: 224×224px, 50kB max)
- [ ] Preview uploaded images in the phone mockup

### 4. Agent List Enhancement

- [ ] Show real agent data from API (status, brand verification state)
- [ ] Agent detail/edit view
- [ ] Delete agent confirmation

### 5. Brand Verification Flow

- [ ] Submit agent for Google verification
- [ ] Display verification status updates
- [ ] Handle questionnaire responses

### 6. Feature Flag Cleanup

- [ ] Re-gate the RCS nav item behind `showRcsOnboarding` once Unleash flag is enabled in production
- [ ] Revert `featureFlagDefaults` to `false` before production commit
- [ ] Uncomment the feature flag check in `pages/u/channels/rcs/index.tsx`

---

## Key Files

### Frontend (burst repo)

| Path | Description |
|---|---|
| `frontend/unified/src/components/RcsOnboarding/AgentCreate/` | Wizard container, component, types, step components |
| `frontend/unified/src/components/RcsOnboarding/AgentList/` | Agent list container, component, types |
| `frontend/unified/src/components/RcsOnboarding/shared/` | PhonePreview component |
| `frontend/unified/src/pages/u/channels/rcs/` | Page routes (index + onboarding) |
| `frontend/unified/src/components/Core/Layout/Menu/items.ts` | Sidebar nav config (RCS item added here) |
| `frontend/unified/src/components/Core/FeatureFlags/featureFlagsConfig.ts` | Feature flag definitions |

### Designs

| Path | Description |
|---|---|
| `v0-designs/` | Original V0 designs (5-step wizard) |
| `v0-designs-NEW/` | Redesigned V0 designs (3-step wizard) — **current** |

---

## Code Conventions

1. **Typography** — Use `@/components/ui/typography/` (`H1`, `H3`, `Paragraph`, `Body1`, etc.)
2. **Import paths** — `@/components/ui/base/X` for shadcn, `@/components/ui/typography/` for text
3. **Feature flags** — Add to `featureFlagsConfig.ts`, gate via `useFeatureFlag()`. Defaults are fallbacks for Unleash errors only.
4. **File formatting** — Tabs, single quotes, semicolons, 100 char width (Prettier)
5. **No `"use client"`** — burst uses Pages Router, not App Router
6. **`@base-ui/react`** — NOT Radix UI. Use `render={<Component />}` instead of `asChild`
7. **Props interfaces** — Always declare at top of component files
8. **Local dev flag bypass** — Unleash overrides defaults; bypass flag directly in page component for testing

---

## Documentation

| Document | Description |
|---|---|
| [Core Functionality](docs/core-functionality.md) | Agent CRUD, test devices, compliance, status lifecycle, API endpoints, DB schema |
| [Implementation Plan](docs/implementation-plan.md) | Tech stack, code governance, V0 design workflow, feature flag strategy, phased build plan |
| [Design Mapping](docs/design-mapping.md) | V0 → burst component mapping, screen-by-screen breakdowns |
| [Backend Integration Plan](docs/backend-integration-plan.md) | RBM partner details, existing backend, agent management API plan, DB schema |
| [Dev Environment](docs/dev-environment.md) | Okteto setup, database access, debugging, useful commands |
| [RBM API Evaluation](docs/api-evaluation-rbm.md) | Google RBM API scored 7.7/10 — selected as provider |
| [Infobip API Evaluation](docs/api-evaluation-infobip.md) | Infobip RCS API scored 6.3/10 — not selected |