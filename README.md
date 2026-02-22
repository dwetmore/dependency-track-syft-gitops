# dependency-track-syft-gitops

GitOps repository for deploying a supply-chain stack on MicroK8s:

- Dependency-Track (frontend + API)
- PostgreSQL (backend database)
- Syft SBOM generation job

This repo is managed by Argo CD using an app-of-apps model.

## What this deploys

| App | Namespace | Source | Purpose |
|---|---|---|---|
| `dependency-track-secrets` | `dependency-track` | local manifests | DB credential secret used by PostgreSQL and Dependency-Track |
| `dependency-track-postgresql` | `dependency-track` | Bitnami Helm chart | Persistent PostgreSQL database |
| `dependency-track` | `dependency-track` | Dependency-Track Helm chart | UI/API for SBOM ingestion and analysis |
| `syft` | `sbom` | local manifests | Scheduled SBOM generation to persistent storage |

## Repository layout

- `root/app-of-apps.yaml`: optional standalone Argo CD bootstrap for this repo.
- `kustomization.yaml`: root kustomization listing child app manifests.
- `apps/`: Argo CD `Application` resources.
- `workloads/dependency-track-secrets/`: DB secret manifest(s).
- `workloads/syft/`: Syft CronJob + PVC manifests.
- `docs/OPERATIONS.md`: day-2 operations runbook.
- `docs/TROUBLESHOOTING.md`: known issues and recovery steps.

## Architecture

1. `dependency-track-secrets` creates `dependency-track-db` secret.
2. `dependency-track-postgresql` consumes that secret and starts PostgreSQL.
3. `dependency-track` connects to PostgreSQL using the same secret.
4. Ingress exposes the UI and `/api` on a single local URL.
5. `syft` CronJob writes SBOM artifacts to a PVC in namespace `sbom`.

## Prerequisites

- MicroK8s with addons:
  - `ingress`
  - `hostpath-storage`
  - `helm3`
- Argo CD installed in namespace `argocd`
- Access to this repository from Argo CD

## Bootstrap

### Option A: Through the existing control plane repo (`argocd-apps`)

If `argocd-apps` already includes `apps/dependency-track-syft-gitops.yaml`, Argo CD will deploy this stack automatically.

### Option B: Standalone bootstrap

```bash
kubectl apply -n argocd -f root/app-of-apps.yaml
```

## Access

### Primary (Ingress)

Current host configured in `apps/dependency-track.yaml`:

- UI + API: `http://dtrack.172.17.93.185.nip.io`

Ingress details:

- `/` -> `dependency-track-frontend:8080`
- `/api` -> `dependency-track-api-server:8080`
- Upload size annotation: `nginx.ingress.kubernetes.io/proxy-body-size: "32m"`

### Fallback (Port-forward)

Use this when ingress/DNS is unavailable.

```bash
kubectl -n dependency-track port-forward svc/dependency-track-frontend 18080:8080
kubectl -n dependency-track port-forward svc/dependency-track-api-server 18081:8080
```

Then open `http://127.0.0.1:18080`.

## Configuration reference

### Dependency-Track app

File: `apps/dependency-track.yaml`

- Chart: `dependency-track` `0.42.0`
- API server PVC: `10Gi` (`microk8s-hostpath`)
- DB mode: external PostgreSQL (`ALPINE_DATABASE_MODE=external`)
- DB credentials: from `dependency-track-db` secret
- Ingress host/path routing and upload size configured via `extraObjects`

### PostgreSQL app

File: `apps/dependency-track-postgresql.yaml`

- Chart: `postgresql` `18.4.0`
- Database/user: `dtrack`
- Password source: `dependency-track-db` secret keys
- PVC: `10Gi` (`microk8s-hostpath`)
- Resource settings (stability tuned):
  - Requests: `cpu 250m`, `memory 512Mi`
  - Limits: `cpu 1`, `memory 1Gi`

### Secrets app

Files:

- `apps/dependency-track-secrets.yaml`
- `workloads/dependency-track-secrets/secret-db.yaml`

Default values are placeholders and must be replaced before production:

- `password`
- `postgres-password`

### Syft app

Files:

- `apps/syft.yaml`
- `workloads/syft/cronjob.yaml`
- `workloads/syft/pvc.yaml`

Defaults:

- Cron schedule: `30 2 * * *`
- Target image: `registry:milvusdb/milvus:v2.5.0-beta`
- Outputs:
  - `/data/milvus_v2.5.0-beta.cdx.json`
  - `/data/milvus_v2.5.0-beta.spdx.json`
- PVC: `syft-sbom-output` (`5Gi`, `microk8s-hostpath`)

## Validation checklist

```bash
kubectl -n argocd get applications | rg 'dependency-track|syft'
kubectl -n dependency-track get pods,svc,ingress,pvc
kubectl -n sbom get cronjob,pvc
```

Expected:

- Argo apps `Synced/Healthy`
- DB/API/frontend pods `Running` and `Ready`
- Ingress host reachable from browser
- SBOM PVC bound

## SBOM upload notes

- UI upload path uses ingress.
- If uploads fail with `413 Request Entity Too Large`, increase `nginx.ingress.kubernetes.io/proxy-body-size`.
- Current value is `32m`.

## Security notes

- `workloads/dependency-track-secrets/secret-db.yaml` currently stores plaintext placeholders for bootstrap.
- For production:
  - move secrets to SealedSecrets/External Secrets/Vault
  - rotate DB credentials and API keys
  - avoid committing real secrets to Git

## Day-2 docs

- Operations runbook: `docs/OPERATIONS.md`
- Troubleshooting guide: `docs/TROUBLESHOOTING.md`
