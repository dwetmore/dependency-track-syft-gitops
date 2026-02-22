# Operations Runbook

This runbook covers common operational tasks for the Dependency-Track + Syft stack.

## 1) Check stack health

```bash
kubectl -n argocd get applications | rg 'dependency-track|syft'
kubectl -n dependency-track get pods,svc,ingress,pvc
kubectl -n sbom get cronjob,job,pvc,pods
```

## 2) Force Argo refresh

```bash
kubectl -n argocd patch application dependency-track-syft-gitops \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

Refresh child apps directly when needed:

```bash
kubectl -n argocd patch application dependency-track --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
kubectl -n argocd patch application dependency-track-postgresql --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
kubectl -n argocd patch application syft --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

## 3) Trigger Syft scan immediately

```bash
kubectl -n sbom create job --from=cronjob/syft-sbom-milvus syft-sbom-manual-$(date +%s)
kubectl -n sbom get jobs,pods
```

Wait for completion:

```bash
kubectl -n sbom wait --for=condition=complete job/<job-name> --timeout=300s
kubectl -n sbom logs job/<job-name>
```

## 4) Verify SBOM artifacts on PVC

```bash
kubectl -n sbom run syft-output-inspect --restart=Never --image=busybox:1.36 \
  --overrides='{"spec":{"volumes":[{"name":"out","persistentVolumeClaim":{"claimName":"syft-sbom-output"}}],"containers":[{"name":"inspect","image":"busybox:1.36","command":["sh","-c","ls -lh /data"],"volumeMounts":[{"name":"out","mountPath":"/data"}]}]}}'
kubectl -n sbom logs syft-output-inspect
kubectl -n sbom delete pod syft-output-inspect --ignore-not-found
```

## 5) Access methods

Ingress URL:

- `http://dtrack.172.17.93.185.nip.io`

Port-forward fallback:

```bash
kubectl -n dependency-track port-forward svc/dependency-track-frontend 18080:8080
kubectl -n dependency-track port-forward svc/dependency-track-api-server 18081:8080
```

## 6) Restart components

```bash
kubectl -n dependency-track rollout restart statefulset/dependency-track-postgresql
kubectl -n dependency-track rollout restart statefulset/dependency-track-api-server
kubectl -n dependency-track rollout restart deployment/dependency-track-frontend
```

## 7) Rotate DB secret (GitOps-managed)

1. Update `workloads/dependency-track-secrets/secret-db.yaml`.
2. Commit and push.
3. Refresh Argo app.
4. Restart PostgreSQL and API server.

Important: If existing DB data was created with older credentials, you may need either:

- password rotation inside PostgreSQL, or
- a DB PVC reset (data destructive).

## 8) Increase upload limit for large SBOMs

Edit `apps/dependency-track.yaml` ingress annotations:

```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "32m"
```

Set a higher value like `100m` if needed, then sync Argo.
