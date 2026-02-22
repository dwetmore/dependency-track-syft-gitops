# Troubleshooting

## Symptom: Login loop (cannot get past login)

Likely causes:

- Frontend points to wrong API URL.
- Ingress routes frontend and API on different hosts without correct config.

Current expected setup:

- Single host ingress with `/api` routed to API service.
- `frontend.apiBaseUrl` set to empty string.

Checks:

```bash
kubectl -n dependency-track get ingress dependency-track-frontend -o yaml
kubectl -n dependency-track get svc dependency-track-frontend dependency-track-api-server
```

## Symptom: Cannot create project

Likely cause:

- Dependency-Track API cannot write to DB because PostgreSQL is unstable.

Checks:

```bash
kubectl -n dependency-track get pods
kubectl -n dependency-track describe pod dependency-track-postgresql-0 | rg -n 'OOMKilled|CrashLoopBackOff|Limits|Requests'
kubectl -n dependency-track logs statefulset/dependency-track-api-server --tail=300
```

Known fix already applied in this repo:

- PostgreSQL resources increased to avoid OOM:
  - request `512Mi`
  - limit `1Gi`

## Symptom: SBOM upload fails with "Request too large"

Cause:

- NGINX ingress request body limit too low.

Fix:

- Increase `nginx.ingress.kubernetes.io/proxy-body-size` in `apps/dependency-track.yaml`.

Current configured value: `32m`.

## Symptom: Dependency-Track API fails DB authentication

Cause:

- DB secret password and actual PostgreSQL user password do not match.

Checks:

```bash
kubectl -n dependency-track get secret dependency-track-db -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n dependency-track logs statefulset/dependency-track-api-server --tail=200 | rg -i 'password authentication failed'
```

Recovery options:

1. Rotate DB user password inside PostgreSQL to match secret.
2. Or reset DB PVC and reinitialize (destructive).

## Symptom: Syft job fails to start with "syft executable not found"

Cause:

- CronJob specifies `command: ["syft", ...]` even though image already defines entrypoint.

Fix:

- Use `args:` only (current repo state already fixed).

## Symptom: Ingress URL not reachable

Cause candidates:

- DNS resolution issue for `nip.io`.
- Host network/WSL routing constraints.

Workaround:

- Use port-forward access.

```bash
kubectl -n dependency-track port-forward svc/dependency-track-frontend 18080:8080
kubectl -n dependency-track port-forward svc/dependency-track-api-server 18081:8080
```
