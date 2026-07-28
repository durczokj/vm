# Week 1 Kubernetes Instructions (Beginner Playbook)

This file is your execution guide for Week 1.
Goal: run one app on Kubernetes on your VM using k3s, expose it, and verify health checks.

## What you will build this week

By the end of this week, you should have:

- a working single-node Kubernetes cluster on your VM (k3s)
- kubectl configured on your laptop
- one deployed app (start with model-api)
- app exposed through Ingress
- health probes enabled

## Learning outcomes

You should understand:

- what a Pod, Deployment, Service, and Ingress do
- why health probes matter
- how to inspect and debug with kubectl

## Day 0: Preparation and safety

### 1. Rotate credentials first

Your previous note included sensitive credentials. Rotate them now before any further setup.

### 2. Confirm your VM is reachable

From your laptop:

```bash
ssh ubuntu@<YOUR_VM_IP>
```

If this fails, fix SSH/networking first.

### 3. Confirm required ports on VM firewall/cloud firewall

Open at minimum:

- 22 (SSH)
- 80 (HTTP)
- 443 (HTTPS)

If you cannot expose 443 this week, continue with 80 and add TLS later.

## Day 1: Install k3s on VM

k3s is a lightweight Kubernetes distribution. It is perfect for learning and single-node setups.

### 1. Install k3s

Run on VM:

```bash
curl -sfL https://get.k3s.io | sh -
```

### 2. Verify k3s service

```bash
sudo systemctl status k3s --no-pager
```

You want to see active (running).

### 3. Verify cluster from VM

```bash
sudo k3s kubectl get nodes
```

Expected: one node with status Ready.

### 4. Why this matters

- k3s installs control plane + worker on one machine.
- You can start learning Kubernetes objects immediately.
- Later, manifests can be reused in GKE with minimal changes.

## Day 2: Configure kubectl on your laptop

You should control the VM cluster from your local machine.

### 1. Install kubectl on macOS

```bash
brew install kubectl
```

### 2. Copy kubeconfig from VM

On your laptop:

```bash
mkdir -p ~/.kube
scp ubuntu@<YOUR_VM_IP>:/etc/rancher/k3s/k3s.yaml ~/.kube/config-k3s
```

### 3. Fix server address in kubeconfig

The copied file usually points to 127.0.0.1. Change it to your VM IP.

```bash
sed -i '' 's/127.0.0.1/<YOUR_VM_IP>/g' ~/.kube/config-k3s
```

### 4. Export kubeconfig and test

```bash
export KUBECONFIG=~/.kube/config-k3s
kubectl get nodes
```

Expected: node status Ready.

### 5. Optional convenience

Add to your shell profile:

```bash
echo 'export KUBECONFIG=~/.kube/config-k3s' >> ~/.zshrc
source ~/.zshrc
```

## Day 3: Deploy first app (model-api)

You will apply 3 Kubernetes objects:

- Deployment: runs and maintains app Pods
- Service: stable internal network endpoint
- Ingress: external HTTP entry point

### 1. Create a workspace folder for manifests

On your laptop (inside your vm repo):

```bash
mkdir -p week1/model-api
```

### 2. Create namespace manifest

Create week1/namespace.yaml:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

### 3. Create deployment manifest

Create week1/model-api/deployment.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-api
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: model-api
  template:
    metadata:
      labels:
        app: model-api
    spec:
      containers:
        - name: model-api
          image: <YOUR_IMAGE>:<TAG>
          ports:
            - containerPort: 8000
          env:
            - name: PORT
              value: "8000"
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 20
```

Replace:

- <YOUR_IMAGE>:<TAG> with your real container image
- port/path if your app differs

### 4. Create service manifest

Create week1/model-api/service.yaml:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: model-api
  namespace: dev
spec:
  selector:
    app: model-api
  ports:
    - port: 80
      targetPort: 8000
      protocol: TCP
  type: ClusterIP
```

### 5. Create ingress manifest

k3s installs Traefik by default, so Ingress should work out of the box.

Create week1/model-api/ingress.yaml:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: model-api
  namespace: dev
spec:
  rules:
    - host: model-api.<YOUR_DOMAIN_OR_IP>.nip.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: model-api
                port:
                  number: 80
```

If you do not have a domain, use nip.io with your VM IP:

- example host: model-api.51.83.199.73.nip.io

## Day 4: Apply manifests and verify

### 1. Apply everything

From your vm repo folder:

```bash
kubectl apply -f week1/namespace.yaml
kubectl apply -f week1/model-api/deployment.yaml
kubectl apply -f week1/model-api/service.yaml
kubectl apply -f week1/model-api/ingress.yaml
```

### 2. Check resources

```bash
kubectl get all -n dev
kubectl get ingress -n dev
```

### 3. Watch rollout

```bash
kubectl rollout status deployment/model-api -n dev
```

Expected: successfully rolled out.

### 4. Test application

```bash
curl -i http://model-api.<YOUR_DOMAIN_OR_IP>.nip.io/health
```

Expected: HTTP 200 (or your health endpoint success response).

## Day 5: Debugging practice

Practice these commands deliberately.

### 1. Get pod names

```bash
kubectl get pods -n dev
```

### 2. Read logs

```bash
kubectl logs -n dev deploy/model-api
```

### 3. Describe pod for events/errors

```bash
kubectl describe pod -n dev <POD_NAME>
```

### 4. Port-forward fallback test

If Ingress has issues, test service directly:

```bash
kubectl port-forward svc/model-api 8080:80 -n dev
curl -i http://localhost:8080/health
```

If this works but Ingress does not, problem is routing/DNS/firewall, not app runtime.

## Troubleshooting guide

### ImagePullBackOff

Cause:
- image tag does not exist
- registry auth missing

Fix:
- verify image and tag
- if private registry, add imagePullSecret

### CrashLoopBackOff

Cause:
- app startup fails
- wrong env vars
- wrong port

Fix:
- inspect logs
- confirm app listens on expected containerPort

### Readiness probe failing

Cause:
- wrong health path or wrong port
- app needs more startup time

Fix:
- correct path/port
- increase initialDelaySeconds

### Ingress unreachable

Cause:
- DNS/host mismatch
- firewall not open
- ingress controller issue

Fix:
- verify host in Ingress
- verify VM firewall 80/443
- run kubectl get pods -n kube-system and check Traefik is healthy

## Day 6: Wrap-up (commit, adapt app repo, decommission old deployment)

Goal by end of day:

- the vm repo is under version control on GitHub
- model-api's own repo has a clean k8s-friendly setup (probes, config, docs)
- the previous non-k8s deployment (durczok.ovh/model_api) is stopped and cleaned up
- only the k3s cluster serves model-api going forward

### 1. Commit the manifests to git

The `week1/` folder is now real infrastructure-as-code. Save it.

Inside `/Users/jakubdurczok/Documents/GitHub/vm`:

```bash
git init            # only if not already initialized
git status
```

Create a `.gitignore` at the repo root to keep secrets out:

```gitignore
# secrets and local notes
vps.txt
vps.rtf
*.pem
*.key
kubeconfig*
.kube/
.DS_Store
```

Then:

```bash
git add .gitignore WEEK1_KUBERNETES_INSTRUCTIONS.md KUBERNETES_LEARNING_PLAN.md week1/
git commit -m "week1: k3s + model-api (deployment, service, ingress) on durczok.ovh"
```

Create a GitHub repo (private):

```bash
gh repo create durczokj/vm --private --source=. --remote=origin --push
```

If you don't have the `gh` CLI, create it via the GitHub web UI, then:

```bash
git remote add origin git@github.com:durczokj/vm.git
git branch -M main
git push -u origin main
```

Sanity check the repo does NOT contain secrets:

```bash
git ls-files | grep -Ei 'vps|secret|token|\.pem|\.key' || echo 'clean'
```

If anything shows up, remove it, add to `.gitignore`, `git rm --cached <file>`, commit, force-push.

### 2. Adapt the model-api repo to the new framework

The model-api repo (`/Users/jakubdurczok/Documents/GitHub/model-api`) should now assume it will be run on Kubernetes, not on the VM directly.

Steps:

1. **Add a versioned image tag and stop using `:latest`.**
   Update the Dockerfile / CI so images are pushed as `durczokj/iris-model-api:vX.Y.Z`.
   Rule of thumb: `:latest` is fine for local dev, never for cluster deploys.

2. **Keep the `/health` route committed.**
   Confirm `src/app.py` in the model-api repo has:

   ```python
   @app.route("/health")
   def health():
       return {"status": "ok"}, 200
   ```

   Commit and push in the model-api repo.

3. **Move k8s manifests closer to the app (optional but recommended).**
   Two valid patterns:
   - Keep manifests in the vm repo (current setup). Good for a mono-infra repo.
   - Add a `deploy/k8s/` folder inside model-api with the same 3 YAMLs. Good for app-owned deploy.

   For Week 1, keep them in the vm repo. Revisit in Week 3 when we introduce Helm/Kustomize.

4. **Pin the image tag in the deployment manifest.**
   In `week1/model-api/deployment.yaml`, change `image: durczokj/iris-model-api:latest` to the versioned tag you just pushed:

   ```yaml
   image: durczokj/iris-model-api:v0.1.0
   ```

   Apply and verify a clean rollout:

   ```bash
   kubectl apply -f week1/model-api/deployment.yaml
   kubectl -n dev rollout status deployment/model-api
   ```

5. **Document the deploy flow in the model-api README.**
   Short section: how to build, push, bump tag, and rollout. Future-you and interviewers will thank you.

### 3. Decommission the old deployment (durczok.ovh/model_api)

The old app was served by a `docker compose` stack directly on the VM (see `docker-compose.yaml` in the model-api repo) and likely fronted by nginx or similar mapping `durczok.ovh/model_api` -> `localhost:3000`.

Do this carefully — you are switching production traffic to Kubernetes.

**Step A: Confirm k8s is serving traffic and users won't notice a gap.**

From your laptop:

```bash
curl -i http://model-api.durczok.ovh/health
curl -s 'http://model-api.durczok.ovh/predict?sepal_length=5.1&petal_length=1.4'
```

Both must succeed before you touch the old deployment.

**Step B: SSH to the VM and discover what's currently serving the old URL.**

```bash
ssh ubuntu@51.83.199.73

# Any docker containers still running?
sudo docker ps

# Any docker compose stacks?
sudo docker compose ls -a

# Anything listening on port 3000 or on 80/443 outside of k3s?
sudo ss -tlnp | grep -E ':80|:443|:3000'

# Any nginx/caddy/apache?
systemctl list-units --type=service --state=running | grep -Ei 'nginx|caddy|apache|httpd'
```

Expected findings:
- k3s owns 80/443 via svclb/traefik. Anything else on those ports is a conflict — but since your ingress works, there is none.
- A docker container `iris-model-api` may still be running on port 3000 (harmless — not exposed publicly, but wasting RAM).
- A reverse proxy (nginx/caddy) may still hold the `/model_api` route on 80/443. Since k3s already claimed 80/443, it either failed silently or isn't running.

**Step C: Stop the old docker-compose stack.**

If `docker compose ls` showed the stack (look for the directory containing the `docker-compose.yaml`), stop it:

```bash
# find where compose was launched
sudo docker compose ls -a
# stop and remove
cd <path shown above>
sudo docker compose down
```

If the stack is gone but a stray container remains:

```bash
sudo docker ps | grep iris
sudo docker rm -f <container-id>
```

**Step D: Remove any reverse proxy config for /model_api.**

If nginx was running:

```bash
sudo nginx -T | grep -i model_api
sudo rm /etc/nginx/sites-enabled/model-api   # or whatever file name
sudo systemctl reload nginx || sudo systemctl disable --now nginx
```

Only disable nginx entirely if nothing else on the VM depends on it.

**Step E: Update DNS to reflect the new reality (optional but tidy).**

Two clean states you can pick from:

1. Keep both hostnames pointing at the VM. `durczok.ovh` still resolves for your website; `model-api.durczok.ovh` is the API. No further DNS change needed.
2. Drop the `/model_api` path from public documentation, portfolio, etc. Update any client that used `https://durczok.ovh/model_api` to use `http://model-api.durczok.ovh` (or `https://` once you add TLS in Week 2).

**Step F: Verify no old artifacts left behind.**

```bash
sudo docker image prune -f
sudo systemctl status k3s --no-pager | head -20
kubectl -n dev get all
```

Free RAM check:

```bash
free -h
```

**Step G: Update Day 6 checklist below.**

### Day 6 checklist

- [ ] vm repo committed and pushed to GitHub
- [ ] `.gitignore` prevents committing vps.txt / kubeconfig / keys
- [ ] model-api image built and pushed with a versioned tag (not `:latest`)
- [ ] `week1/model-api/deployment.yaml` pins that versioned tag
- [ ] rollout succeeded with the pinned tag
- [ ] old docker compose / container for iris-model-api stopped
- [ ] old nginx (or similar) route for `/model_api` removed
- [ ] `curl http://model-api.durczok.ovh/health` still returns 200
- [ ] `free -h` shows recovered memory

## End-of-week checklist

Mark complete when done:

- [ ] k3s installed and node Ready
- [ ] kubectl from laptop connected to cluster
- [ ] namespace dev created
- [ ] model-api deployed as Deployment + Service
- [ ] ingress route working
- [ ] readiness/liveness probes passing
- [ ] you can debug using logs and describe
- [ ] Day 6 wrap-up complete (git, versioned image, old deployment removed)

## Stretch goal (optional)

Add a second app (chains) using the same pattern in a new folder:

- week1/chains/deployment.yaml
- week1/chains/service.yaml
- week1/chains/ingress.yaml

This reinforces the deployment workflow before Week 2.

## Notes template

At the end of each session, write 3 bullets:

1. What I changed
2. What broke
3. How I fixed it

This habit is extremely valuable for fast growth and interview stories.
