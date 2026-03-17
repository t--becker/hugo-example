# RCS Onboarding Project

## Project Overview

Adding RCS (Rich Communication Services) onboarding into the Burst/Kudosity application.


The development environment runs on **Okteto** — a hosted Kubernetes environment at `kudosity.okteto.dev`. The burst monorepo is at `/Users/tbc/repo/burst`.

---

## Developer Environment Setup

### Prerequisites

- Git and the burst monorepo cloned to `/Users/tbc/repo/burst`
- Okteto CLI installed (`brew install okteto` or see [Okteto docs](https://www.okteto.com/docs/))
- `kubectl` installed
- GitHub account with access to the burst monorepo

### One-time Setup Steps

#### 1. Set Okteto Context

```bash
okteto context use https://kudosity.okteto.dev
```

This authenticates via GitHub and sets your CLI context to the Kudosity Okteto cluster. Your namespace is based on your GitHub username (e.g. `t-becker`).

#### 2. Generate Kubeconfig

```bash
okteto kubeconfig
```

Updates `~/.kube/config` with the context `kudosity_okteto_dev/<your-namespace>`. This enables `kubectl` and tools like Lens to access your cluster.

#### 3. Deploy the Dev Environment (Web Portal)

1. Go to [kudosity.okteto.dev](https://kudosity.okteto.dev)
2. Click **Deploy Dev Environment**
3. Select **GitHub** → choose the **burst** monorepo
4. Select your branch (e.g. `develop`)
5. Click **Launch**

This takes ~15–20 minutes to build and deploy all services.

#### 4. Verify

```bash
kubectl get pods -n <your-namespace>
```

All ~51 services should be `Running`.

---

## Current Environment Status (as of setup)

- **Namespace**: `t-becker` @ `kudosity.okteto.dev`
- **Kubeconfig context**: `kudosity_okteto_dev/t-becker`
- **51 pods running** — full monorepo deployed
- `notification-domain` — running but readiness probe returning 503 (non-blocking for most development)

---

## Daily Development Workflow

### Enter Dev Mode for a Domain

Navigate to the domain's `_domain` folder and run `okteto up`:

```bash
cd /Users/tbc/repo/burst/backend/rcs/rcs_domain
okteto up
```

This replaces the remote container with a dev image that syncs your local filesystem and runs `modd` to watch for code changes.

To run additional commands in the dev container:

```bash
okteto exec rcs-domain -- bash
```

### Make API Requests

All ingresses include the namespace in the hostname:

```
https://api-t-becker.kudosity.okteto.dev/v2/...
```

Get your `x-api-key` from [staging.burstage.com/profile](https://staging.burstage.com/profile) → SETTINGS → scroll to API key.

Example:
```bash
curl --location 'https://api-t-becker.kudosity.okteto.dev/v2/sender' \
  --header 'x-api-key: <your-api-key>'
```

### Build & Deploy a Domain

```bash
# Redeploy using existing image
cd /Users/tbc/repo/burst/backend/rcs/rcs_domain
okteto deploy

# Build new image and deploy
okteto deploy --build
```

### Rebuild the Okteto Base Images

```bash
cd /Users/tbc/repo/burst/ops/okteto
okteto build
```

---

## Database Access

### PostgreSQL

```bash
okteto context use https://kudosity.okteto.dev
okteto kubeconfig
kubectl port-forward svc/postgresql 15432:5432
PGPASSWORD=admin psql --host localhost --port 15432 --user postgres
```

### MongoDB (via Lens or kubectl)

```bash
kubectl port-forward svc/mongodb 27017:27017
```

Then connect with MongoDB Compass:
```
mongodb://root:root123@localhost:27017/?authSource=admin&directConnection=true&ssl=false
```

### Database Fixtures

```bash
cd /Users/tbc/repo/burst/ops/okteto

# Inject fixtures (erase first if re-running)
okteto up -e cmd=erase-dbdata
okteto up -e cmd=inject-dbdata

# Dump data from environment
okteto up -e cmd=dump-dbdata
```

### Database Migrations

```bash
# Bring up the dev container for the domain
cd /Users/tbc/repo/burst/backend/rcs/rcs_domain
okteto up

# In another shell, run migration
okteto exec rcs-domain -- ../run_migration.sh rcs
```

---

## gRPC / RPC Requests

gRPC endpoints are not externally exposed — use port-forwarding:

```bash
kubectl get pods -n t-becker
kubectl port-forward pods/<rcs-domain-pod-name> 5001:5000 --namespace=t-becker
```

Then with `grpcurl`:
```bash
grpcurl -plaintext localhost:5001 list
```

Or with `evans`:
```bash
evans -r --host localhost --port 5001
```

---

## Debugging with Delve

Add a port forward to the domain's `okteto.yml`:

```yaml
dev:
  rcs-domain:
    forward:
      - 2345:2345
```

Then:
```bash
okteto up --command bash
# Inside the container:
go build -gcflags "all=-N -l" -o /tmp/app .
dlv --listen=:2345 --headless=true --accept-multiclient --api-version=2 exec /tmp/app
```

Connect your IDE (GoLand/VSCode) to `localhost:2345` as a remote debugger.

---

## Useful Commands Reference

```bash
# Check pod status
kubectl get pods -n t-becker

# List services and ports
kubectl get services -n t-becker

# View logs for a pod
kubectl logs -n t-becker <pod-name> --follow

# Port forward any service
kubectl port-forward svc/<service-name> <local-port>:<remote-port> -n t-becker

# Destroy a domain (then redeploy to pick up env var changes)
cd /path/to/domain/_domain
okteto destroy

# Hard reset (create a new namespace)
okteto namespace create <new-name>
okteto namespace use <new-name>

# Re-run kubeconfig after namespace switch
okteto kubeconfig
```

---

## Troubleshooting

### `okteto context list` returns error
Run `okteto context use https://kudosity.okteto.dev` to re-authenticate.

### "Couldn't create a pid file" on `okteto up`
```bash
sudo chown -R $(whoami): ~/.okteto
```
Or check/increase your inotify watcher limit.

### Env var changes not picked up
Destroy the domain before redeploying:
```bash
okteto destroy
okteto deploy
```
Verify with: `kubectl describe configmap rcs-domain -n t-becker`

### Preview Environments
Add the label `preview` to a GitHub PR to automatically spin up a dedicated namespace for that PR. Endpoints are posted as a PR comment.

---

## Key URLs

| Service | URL |
|---|---|
| Okteto portal | https://kudosity.okteto.dev |
| Your namespace | https://kudosity.okteto.dev/spaces/t-becker |
| API (your namespace) | https://api-t-becker.kudosity.okteto.dev |
| Jaeger tracing | Available via Okteto portal service links |
| pgAdmin | Available via Okteto portal service links |
| MailDev | Available via Okteto portal service links |
| RabbitMQ UI | Available via Okteto portal service links |

---

## Documentation

| Document | Description |
| --- | --- |
| [Core Functionality](docs/core-functionality.md) | V0-identified core functionality — Agent CRUD, test devices, compliance, status lifecycle, API endpoints, DB schema |
| [Infobip API Evaluation](docs/api-evaluation-infobip.md) | Infobip RCS API scored against core requirements — 6.3/10 overall, strong on messaging/templates, weak on compliance/lifecycle |
| [RBM API Evaluation](docs/api-evaluation-rbm.md) | Google RBM API scored against core requirements — 7.7/10 overall, strong lifecycle management, no agent cap, full API coverage |
| [Implementation Plan](docs/implementation-plan.md) | Full build plan — tech stack, code governance, V0 design workflow, feature flag strategy, 5-phase build plan |
| [Design Mapping](docs/design-mapping.md) | V0 → burst component mapping, PR plan, screen-by-screen breakdowns, adaptation rules |
| [Backend Integration Plan](docs/backend-integration-plan.md) | RBM partner details, existing backend architecture, agent management API plan, new files/routes/DB schema |

---

## Current Branch & PR Status

| Branch | PR | Status | Description |
|---|---|---|---|
| `feat/rcs-onboarding-shell` | [#9155](https://github.com/burstsms/burst/pull/9155) | Open | Feature flag + empty shell (2 commits, CodeRabbit feedback addressed) |
| `feat/rcs-onboarding-shell` | PR 2 (not yet opened) | In Progress | Agent List + Agent Create wizard (new 3-step design) + RCS sidebar nav |

**Working directory:** `burst` repo is on `feat/rcs-onboarding-shell` branch.

---

## Code Conventions Learned

These are conventions discovered during implementation that a new session should follow:

1. **Typography** — Use `@/components/ui/typography/` (`H1`, `H3`, `Paragraph`, `Body1`, etc.) instead of raw `<h1>`, `<p>` elements
2. **Props interfaces** — Always declare a `Props` interface at the top of component files, even if empty
3. **Conditional rendering** — Use ternaries for simple two-way branches; use `&&` for gated rendering
4. **Import paths** — `@/components/ui/base/X` for shadcn components, `@/components/ui/typography/` for text
5. **Feature flags** — Add to `featureFlagsConfig.ts` with default `false`, gate via `useFeatureFlag()` hook
6. **File formatting** — Tabs, single quotes, semicolons, 100 char width (Prettier)
7. **No `"use client"`** — burst uses Pages Router, not App Router
8. **Missing shadcn components** — `checkbox` and `collapsible` need to be added for PR 5 (Compliance step)
9. **`@base-ui/react` composition** — burst's shadcn components use `@base-ui/react`, NOT Radix UI. Use `render={<Component />}` instead of `asChild` for element composition (e.g. `<DropdownMenuTrigger render={<Button />} />`)
10. **Feature flag defaults vs Unleash** — `featureFlagsConfig.ts` defaults are only fallbacks when Unleash errors or disconnects. When Unleash is connected, the server value overrides local defaults. For local testing, bypass the flag directly in the page component.
11. **Agent List uses only existing components** — `Button`, `Card`, `Badge`, `DropdownMenu`, typography (`H1`, `H2`, `H3`, `Body1`, `Paragraph`) — all from `@/components/ui/base/` and `@/components/ui/typography/`. No new design system components were created.

---

## Next Steps

- [x] Define core functionality requirements
- [x] Evaluate RBM vs InfoBip API capabilities — **Decision: RBM selected**
- [x] Write implementation plan
- [x] Download V0 designs into `/v0-designs/` folder
- [x] Read & map V0 designs to burst component library → `docs/design-mapping.md`
- [x] Implement Phase 1a: Feature flag + empty shell — [PR #9155](https://github.com/burstsms/burst/pull/9155)
- [x] Implement Phase 1b: Agent List page with mock data
- [x] Implement Agent Create Wizard — **redesigned to 3-step flow** (new designs in `v0-designs-NEW/`)
  - Step 1: Sender Details (name, brand, region, billing category, use case)
  - Step 2: Public Profile (display name, description, color, contacts, images, URLs + phone preview)
  - Step 3: Test & Preview (test devices table, add device, send test message)
- [x] Add RCS navigation link to sidebar (Channels → RCS)
- [ ] Add form validation (required fields, URL format, phone format, character limits)
- [ ] Implement image upload for banner/logo (with size/dimension validation)
- [ ] Implement Phase 2: Backend — Agent management Google service + CRUD API routes + DB migration
- [ ] Implement Phase 3: Wire frontend to backend API (replace mock data with real API calls)
- [ ] Implement Phase 4: Agent verification & launch flow
