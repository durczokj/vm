# vm

Kubernetes manifests for the k3s cluster on `durczok.ovh` (`51.83.199.73`).

## Layout

```
apps/                  # per-app platform manifests
  model-api/           # https://model-api.durczok.ovh
  portfolio/           # https://portfolio.durczok.ovh
platform/              # cluster-wide infrastructure
  namespaces.yaml
```

Each `apps/<name>/` directory holds the platform-owned objects for that app: `Service`, `Ingress`, and (when needed) `ConfigMap` and Traefik `Middleware`. The app's `Deployment` lives in the **app's own repo** under `deploy/k8s/deployment.yaml` and is applied by the app's release CI.

`platform/` holds shared, cluster-scoped resources (namespaces now; `ClusterIssuer` and shared middlewares to come).

## Ownership

| Object | Repo |
|---|---|
| `Namespace` | this repo — `platform/namespaces.yaml` |
| `Service` | this repo — `apps/<name>/service.yaml` |
| `Ingress` | this repo — `apps/<name>/ingress.yaml` |
| `ConfigMap` | this repo — `apps/<name>/configmap.yaml` |
| `Middleware` | this repo — `apps/<name>/ingress.yaml` |
| `Deployment` | the app's repo — `deploy/k8s/deployment.yaml` |
| `Secret` | applied out-of-band with `kubectl create secret …` |

Rationale: platform objects change rarely; Deployments change on every release. One owner per object, no drift.

## Release flow (per app)

1. Cut a GitHub Release in the app's repo.
2. The app's `deploy.yaml` workflow:
   - Builds `durczokj/<name>:<tag>` and pushes to Docker Hub.
   - Renders `deploy/k8s/deployment.yaml` (`__IMAGE_TAG__` → `<tag>`).
   - `scp`s the rendered manifest to the VM and runs `kubectl apply` + `rollout status`.

## Bootstrap

```bash
kubectl apply -f platform/
kubectl apply -f apps/model-api/
kubectl apply -f apps/portfolio/
```

## Common operations

```bash
# what's running
kubectl -n dev get deploy,svc,ingress

# which image is live
kubectl -n dev get deploy <name> -o jsonpath='{.spec.template.spec.containers[0].image}'; echo

# roll back
kubectl -n dev rollout undo deployment/<name>

# logs
kubectl -n dev logs deploy/<name> --tail=200 -f
```

## Access

- Kubeconfig lives at `~/.kube/config-k3s` on the operator's machine (never committed).
- SSH to the VM: `ssh -i ~/.ssh/ovh_deploy ubuntu@51.83.199.73`.
