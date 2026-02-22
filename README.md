# dependency-track-syft-gitops

GitOps repository for deploying Dependency-Track and Syft to MicroK8s with Argo CD.

## Components

- `dependency-track-postgresql`: PostgreSQL backing database for Dependency-Track.
- `dependency-track`: Dependency-Track API + frontend.
- `syft`: CronJob that generates SBOMs and writes them to a persistent volume.

## Layout

- `root/app-of-apps.yaml`: Optional bootstrap Application for this repo.
- `kustomization.yaml`: Includes all Argo CD child applications.
- `apps/*.yaml`: Argo CD Application resources.
- `workloads/syft`: Kubernetes manifests for Syft workload.

## Bootstrap (standalone)

```bash
kubectl apply -n argocd -f root/app-of-apps.yaml
```

## Access Dependency-Track (local)

This deployment is configured for local port-forward access.

```bash
kubectl -n dependency-track port-forward svc/dependency-track-frontend 18080:8080
kubectl -n dependency-track port-forward svc/dependency-track-api-server 18081:8080
```

Then open:

- UI: `http://127.0.0.1:18080`
- API: `http://127.0.0.1:18081/api/version`

## Notes

- Database and API secret values in `apps/*.yaml` are defaults for bootstrap. Replace them for your environment.
- Syft defaults to scanning `milvusdb/milvus:v2.5.0-beta` and writes CycloneDX + SPDX JSON to PVC.
