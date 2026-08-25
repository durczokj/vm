# RUNBOOK

Operational reference for the `vm` k3s cluster. Read this when something is on fire
or when you're about to do something you rarely do. Not a tutorial — that's
[`notebooks/ARCHITECTURE.md`](notebooks/ARCHITECTURE.md).

- **Cluster:** single-node k3s on OVH VPS `51.83.199.73`
- **Namespaces:** `prod` (all 5 apps), `dev` (model-api + vendor-manager)
- **Ingress:** Traefik (bundled with k3s)
- **TLS:** cert-manager + Let's Encrypt HTTP-01, per-hostname certs
- **Manifests source of truth:** this repo (`platform/`, `apps/*/base/`, `overlays/*/`)
- **App image manifests:** app repos, deployed via `vm/.github/actions/deploy-to-k3s`

---

## 1. Deploy flow (who ships what, on which event)

### Platform (namespaces, cluster-issuer, ingresses, configmaps, services, StatefulSets)

Owner: this repo. Workflow: `.github/workflows/apply-platform.yaml`.

| Event                                          | Result                          |
|------------------------------------------------|---------------------------------|
| `release` published, `prerelease=false`        | `kubectl apply -k overlays/prod`|
| `release` published, `prerelease=true`         | `kubectl apply -k overlays/dev` |
| `workflow_dispatch` (`env=prod\|dev\|both`)    | diff or apply the chosen overlay|

Cut a full release on `vm` to promote platform changes to prod. Cut a pre-release
to test the dev overlay first. Use `workflow_dispatch` with `mode=diff` any time
to preview a change without applying.

### Apps (stateless Deployments: model-api, vendor-manager, chains, chains-student, portfolio)

Owner: each app repo (`model-api/`, `vendor_manager/`, `chains/`, …). Workflow: that repo's
`Build and Deploy` calling `durczokj/vm/.github/actions/deploy-to-k3s@main`.

| Event                                    | Result                              |
|------------------------------------------|-------------------------------------|
| `release` published, `prerelease=false`  | Build image, deploy to **prod**     |
| `release` published, `prerelease=true`   | Build image, deploy to **dev**      |
| `workflow_dispatch` (`namespace` input)  | Build image, deploy to chosen ns    |

Only `model-api` and `vendor-manager` have both dev and prod slots today. The
other three ship straight to prod.

---

## 2. Everyday commands

Assumes `kubectl` context points at the k3s VM (or you SSH in and run `sudo k3s kubectl`).

```bash
# Where is everything?
kubectl get ns
kubectl -n prod get deploy,pod,ingress,certificate
kubectl -n dev  get deploy,pod,ingress,certificate

# Preview a platform overlay change without applying
kubectl diff -k overlays/prod
kubectl diff -k overlays/dev

# Apply an overlay by hand (normally CI does this)
kubectl apply -k overlays/prod
kubectl apply -k overlays/dev

# Follow a rollout
kubectl -n prod rollout status deploy/vendor-manager --timeout=180s

# Tail pod logs
kubectl -n prod logs deploy/vendor-manager -f --tail=100

# Describe when the pod is unhappy
kubectl -n dev describe pod <podname>

# Get an interactive shell in a running container
kubectl -n dev exec -it deploy/vendor-manager -- /bin/sh
```

---

## 3. In-cluster DNS cheat sheet

Every Service gets three names in CoreDNS. Verified 2026-08-20 in this cluster:

| From namespace | URL                                                  | Result | Notes                                   |
|----------------|------------------------------------------------------|--------|-----------------------------------------|
| `prod`         | `http://model-api/health`                            | 200    | short form; same-namespace only         |
| `prod`         | `http://model-api.prod.svc.cluster.local/health`     | 200    | FQDN; works from any namespace          |
| `dev`          | `http://model-api.prod.svc.cluster.local/health`     | 200    | cross-namespace via FQDN                |
| `dev`          | `http://model-api/health`                            | 200    | dev also has its own `model-api` Service|

Rules:

- `<svc>` — resolves only within the caller's own namespace (via CoreDNS search domain).
- `<svc>.<ns>` — resolves across namespaces; shorter than the full FQDN.
- `<svc>.<ns>.svc.cluster.local` — fully qualified; the only form that is stable no matter
  where the caller pod sits. **Use this when you're not sure.**

To reproduce the demo without `curl` in the app image, use Python's stdlib:

```bash
kubectl -n prod exec deploy/vendor-manager -c vendor-manager -- \
  python -c "import urllib.request; r=urllib.request.urlopen('http://model-api/health',timeout=5); print(r.status)"
```

---

## 4. Deploying a specific tag to dev (self-service iteration)

```bash
cd /Users/jakubdurczok/Documents/GitHub/model-api  # or vendor_manager, chains, …
gh workflow run "Build and Deploy" -f tag=v0.1.5 -f namespace=dev
gh run watch $(gh run list --limit 1 --json databaseId --jq '.[0].databaseId')
kubectl -n dev rollout status deploy/model-api
```

Same for prod — pass `-f namespace=prod`. Do this only when you want to bypass the
release-based promotion flow (usually you don't).

---

## 5. Rollback

### Rollback the last app deploy (fast, in-cluster)

```bash
kubectl -n <ns> rollout history deploy/<name>
kubectl -n <ns> rollout undo    deploy/<name>
kubectl -n <ns> rollout status  deploy/<name>
```

This flips the running pods back to the previous ReplicaSet's image without
touching git. Fine for a quick fix; **still cut a proper release afterward** to
resync the "what's actually running" with "what git says is deployed".

### Rollback via redeploy of the previous release tag

```bash
cd /Users/jakubdurczok/Documents/GitHub/<app>
gh workflow run "Build and Deploy" -f tag=<previous-tag> -f namespace=<ns>
```

Idempotent; same as any other deploy of that tag.

### Rollback a platform change

Platform overlays are managed by the `apply-platform.yaml` workflow. Revert the
offending commit on `main` and cut a new release (or `workflow_dispatch` with
`mode=apply env=prod`). Kustomize `apply -k` is idempotent — it converges the
cluster to whatever the checkout says.

---

## 6. Common failure modes

### Pod stuck in `Init:CrashLoopBackOff` immediately after deploy

```bash
kubectl -n <ns> logs <pod> -c <init-container-name>
```

Typical causes:
- **Missing / wrong Secret key** — a Django app reads `env_required("DATABASE_PASSWORD")`
  but the Secret was created with `DATABASE_URL`. Fix the Secret, don't invent a new key.
- **Migrate can't reach the DB** — the DB StatefulSet isn't Ready yet. Wait or check
  its pod/PVC.

### Main container starts, then gets SIGTERM ~1 min later

Almost always readiness/liveness probes failing. Check the logs of the container
(not the init container):

```bash
kubectl -n <ns> logs <pod> -c <container> --tail=50 --previous
```

Look for `DisallowedHost`, `502`, connection refused, etc. Known gotcha:
`vendor-manager`'s probes hardcode `Host: vendor-manager.durczok.ovh`, so if you
add a new environment its ConfigMap `DJANGO_ALLOWED_HOSTS` must include the prod
hostname too. See `overlays/dev/vendor-manager-configmap-patches.yaml`.

### Cert stuck `Ready=False` for > 5 min

```bash
kubectl -n <ns> describe certificate <name>-tls
kubectl -n <ns> get order,challenge
kubectl -n <ns> describe challenge
```

Two common causes:
1. **DNS record missing.** `*.durczok.ovh` is NOT a wildcard — you must add an A
   record for each new hostname in the OVH DNS console pointing at `51.83.199.73`.
2. **CoreDNS cached NXDOMAIN.** cert-manager does a self-check before telling
   Let's Encrypt to validate; if CoreDNS resolved the hostname while it didn't
   exist yet, it's cached negative. Fix:
   ```bash
   kubectl -n kube-system rollout restart deployment coredns
   kubectl -n <ns> delete certificate <name>-tls
   # kubectl re-creates it from the Ingress annotation and re-tries fresh.
   ```

### `Endpoints` for a Service is empty (503 through the ingress)

Selector/label mismatch. Usually caused by `commonLabels` in Kustomize writing
into `spec.selector`. Use the newer `labels: [{includeSelectors: false, pairs: …}]`
form instead. Verify:

```bash
kubectl -n <ns> get endpoints <svc>
```

If the ADDRESSES column is empty and pods are Running, the selector is wrong.

### PVC won't bind

k3s's `local-path` provisioner uses `WaitForFirstConsumer` — the PVC only binds
once a pod schedules against it. `kubectl get pvc` says `Pending` until then.
That's normal, not broken.

---

## 7. Manual DB operations

### Snapshot a prod DB

```bash
kubectl -n prod exec vendor-manager-db-0 -- \
  pg_dump -U vendor_manager -d vendor_manager -Fc > vendor_manager-$(date +%F).dump
```

`-Fc` is compressed custom format; safe across minor versions and restorable
into either a fresh dev DB or a lab.

### Restore into dev

```bash
kubectl -n dev exec -i vendor-manager-db-0 -- \
  pg_restore -U vendor_manager -d vendor_manager --clean --if-exists < vendor_manager-2026-08-20.dump
```

Do this *before* first deploy, otherwise the Django `migrate` init container
runs against an empty schema and diverges.

Scheduled backups are Week 6 work.

---

## 8. Secrets

Secrets are imperative today (not in git). Bootstrap for a new environment or
after rotating:

```bash
# vendor-manager DB password (shared between the Postgres StatefulSet and the app)
DB_PASS=$(openssl rand -base64 24)
kubectl -n <ns> create secret generic vendor-manager-db-secret --from-literal=POSTGRES_PASSWORD="$DB_PASS"
kubectl -n <ns> create secret generic vendor-manager-secrets \
  --from-literal=DATABASE_PASSWORD="$DB_PASS" \
  --from-literal=DJANGO_SECRET_KEY="$(python -c 'import secrets;print(secrets.token_urlsafe(60))')"
```

**Do not** invent keys like `DATABASE_URL` — the app's `settings.py` reads
`DATABASE_PASSWORD`. Adding an unused `DATABASE_URL` won't help and will silently
mask the real problem.

Record the values in `credentials.txt` in the existing style. Sealed Secrets is
Week 6.

---

## 9. Adding a new hostname

1. In the OVH DNS console, add an `A` record `<name>` → `51.83.199.73`, TTL 60.
2. Wait for `dig +short @1.1.1.1 <name>.durczok.ovh` to return the IP.
3. `kubectl -n kube-system rollout restart deployment coredns` (flush any negative cache).
4. Create/patch the Ingress with the new host and a cert-manager annotation.
5. Watch `kubectl -n <ns> get certificate` until Ready.

---

## 10. vendor_manager MCP server

The MCP server (P11) is a thin FastMCP wrapper around the vendor_manager REST
API. It runs alongside the main app and shares its release tag.

**Hostnames**

| Overlay | Host                                | Backend Service (namespace)      |
|---------|-------------------------------------|----------------------------------|
| prod    | `vendor-manager-mcp.durczok.ovh`    | `vendor-manager-mcp` in `prod`   |
| dev     | `mcp-dev.durczok.ovh`               | `vendor-manager-mcp` in `dev`    |

**In-cluster API URL** (set via `VM_API_BASE_URL` in
`apps/vendor-manager-mcp/base/configmap.yaml`):

* prod → `http://vendor-manager.prod.svc.cluster.local/api/v1`
* dev  → `http://vendor-manager.dev.svc.cluster.local/api/v1`
  (patched in `overlays/dev/vendor-manager-mcp-configmap-patches.yaml`)

**Auth model.** The MCP server holds no credentials. It forwards the
`Authorization: Basic …` header from the incoming MCP request to Django on every
outbound call. RBAC is enforced entirely inside vendor_manager
(`accessible_to(user)` querysets).

**DNS.** Add A records `vendor-manager-mcp.durczok.ovh` and
`mcp-dev.durczok.ovh` → `51.83.199.73` in OVH, then follow the standard
"adding a new hostname" flow above.

**Traefik guardrails.** The ingress references
`vendor-manager-mcp-guardrails@kubernetescrd` — a Middleware **chain** that
combines a 60 req/min rate-limit (burst 20) and a 1 MB request-body cap. Verify
with:

```bash
for i in $(seq 1 70); do
  curl -s -o /dev/null -w '%{http_code}\n' \
    https://mcp-dev.durczok.ovh/healthz
done | sort | uniq -c
```

The 61st request within a minute must return `429`.

**Tool surface.** The MCP server ships two tools: `describe_api` and
`vm_api_request`. `describe_api` returns the OpenAPI schema so the LLM can
discover every endpoint the REST API exposes; `vm_api_request` then lets it
call any of them (`GET`, `POST`, `PATCH`, `DELETE`, …). Django's RBAC is
enforced on every outbound call, so the LLM only sees what the caller's
Basic credentials allow. There is no separate `list_cost_lines` /
`list_entity_options` tool — those endpoints are reachable through
`vm_api_request` at `/api/v1/dashboards/cost-lines/` and
`/api/v1/dashboards/entity-options/`.

**Rollout order on a release.** vendor_manager's `Build and Deploy` workflow
builds both `durczokj/vendor-manager:<tag>` and
`durczokj/vendor-manager-mcp:<tag>` from the same commit, then applies
`deploy/k8s/deployment.yaml` **before** `deploy/k8s/deployment-mcp.yaml`. If the
MCP rollout hangs, check `kubectl -n <ns> logs deploy/vendor-manager-mcp
--tail=100` for `VM_API_BASE_URL` errors or Django 5xx.
