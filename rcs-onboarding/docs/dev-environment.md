# Developer Environment Setup

> Reference guide for Okteto, database access, debugging, and useful commands.

---

## Prerequisites

- Git and the burst monorepo cloned to `/Users/tbc/repo/burst`
- Okteto CLI installed (`brew install okteto` or see [Okteto docs](https://www.okteto.com/docs/))
- `kubectl` installed
- GitHub account with access to the burst monorepo

## One-time Setup Steps

### 1. Set Okteto Context

```bash
okteto context use https://kudosity.okteto.dev
```

### 2. Generate Kubeconfig

```bash
okteto kubeconfig
```

### 3. Deploy the Dev Environment (Web Portal)

1. Go to [kudosity.okteto.dev](https://kudosity.okteto.dev)
2. Click **Deploy Dev Environment**
3. Select **GitHub** → choose the **burst** monorepo
4. Select your branch (e.g. `develop`)
5. Click **Launch** (takes ~15–20 minutes)

### 4. Verify

```bash
kubectl get pods -n <your-namespace>
```

---

## Current Environment Status

- **Namespace**: `t-becker` @ `kudosity.okteto.dev`
- **Kubeconfig context**: `kudosity_okteto_dev/t-becker`
- **51 pods running** — full monorepo deployed

---

## Daily Development Workflow

### Enter Dev Mode for a Domain

```bash
cd /Users/tbc/repo/burst/backend/rcs/rcs_domain
okteto up
```

### Make API Requests

```
https://api-t-becker.kudosity.okteto.dev/v2/...
```

Get your `x-api-key` from [staging.burstage.com/profile](https://staging.burstage.com/profile) → SETTINGS → scroll to API key.

### Build & Deploy a Domain

```bash
cd /Users/tbc/repo/burst/backend/rcs/rcs_domain
okteto deploy          # Redeploy using existing image
okteto deploy --build  # Build new image and deploy
```

---

## Database Access

### PostgreSQL

```bash
kubectl port-forward svc/postgresql 15432:5432
PGPASSWORD=admin psql --host localhost --port 15432 --user postgres
```

### MongoDB

```bash
kubectl port-forward svc/mongodb 27017:27017
# Connect: mongodb://root:root123@localhost:27017/?authSource=admin&directConnection=true&ssl=false
```

### Database Fixtures

```bash
cd /Users/tbc/repo/burst/ops/okteto
okteto up -e cmd=erase-dbdata
okteto up -e cmd=inject-dbdata
```

### Database Migrations

```bash
cd /Users/tbc/repo/burst/backend/rcs/rcs_domain
okteto up
# In another shell:
okteto exec rcs-domain -- ../run_migration.sh rcs
```

---

## gRPC / RPC Requests

```bash
kubectl port-forward pods/<rcs-domain-pod-name> 5001:5000 --namespace=t-becker
grpcurl -plaintext localhost:5001 list
```

---

## Debugging with Delve

```yaml
# Add to domain's okteto.yml:
dev:
  rcs-domain:
    forward:
      - 2345:2345
```

```bash
okteto up --command bash
go build -gcflags "all=-N -l" -o /tmp/app .
dlv --listen=:2345 --headless=true --accept-multiclient --api-version=2 exec /tmp/app
```

---

## Useful Commands

```bash
kubectl get pods -n t-becker
kubectl get services -n t-becker
kubectl logs -n t-becker <pod-name> --follow
kubectl port-forward svc/<service-name> <local-port>:<remote-port> -n t-becker
okteto destroy   # Then redeploy to pick up env var changes
```

---

## Troubleshooting

- **`okteto context list` error** → `okteto context use https://kudosity.okteto.dev`
- **"Couldn't create a pid file"** → `sudo chown -R $(whoami): ~/.okteto`
- **Env var changes not picked up** → `okteto destroy` then `okteto deploy`
- **Preview Environments** → Add label `preview` to a GitHub PR

---

## Key URLs

| Service | URL |
|---|---|
| Okteto portal | https://kudosity.okteto.dev |
| Your namespace | https://kudosity.okteto.dev/spaces/t-becker |
| API (your namespace) | https://api-t-becker.kudosity.okteto.dev |