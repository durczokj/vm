# Kubernetes Learning Plan (Beginner -> Job Ready)

This plan is implementation-first. You will build real deployments from simple to more advanced, using your own projects.

## Step 0: Security First (Do this now)

- Rotate all exposed credentials and SSH keys immediately.
- Store secrets in a password manager.
- Never keep private keys or passwords in project files.

## Goal

By the end of this plan, you will be able to:

- Deploy multiple services on Kubernetes.
- Manage environments with a centralized infrastructure repo.
- Use GitOps for safe, repeatable deployments.
- Operate a cluster with observability and basic security controls.

## Your Project Order

Deploy in this order (easy to hard):

1. model-api
2. chains
3. chains-student
4. portfolio
5. vendor_manager

## Phase 1: Super Simple Start (Week 1)

### Outcome
Run one app in Kubernetes on a single VM.

### Tasks

1. Install a local cluster on VM using k3s.
2. Install kubectl on your machine.
3. Deploy one app with a Deployment and Service.
4. Expose it with Ingress.
5. Add basic health checks.

### Deliverables

- One running app accessible via domain or IP.
- A folder with Kubernetes YAML for that app.

## Phase 2: Standardize Containers (Week 2)

### Outcome
All apps can build and run as production-like containers.

### Tasks

1. Add/clean Dockerfile for each repo.
2. Add a health endpoint in each service.
3. Standardize startup command and port config.
4. Push images to a container registry.

### Deliverables

- All apps produce container images.
- Versioned tags for each build.

## Phase 3: Multi-Service Kubernetes (Week 3)

### Outcome
Run all services in one cluster with clean routing.

### Tasks

1. Create namespaces: dev, stage.
2. Deploy each app into dev.
3. Configure Ingress routes per app.
4. Add ConfigMap and Secret per service.

### Deliverables

- All services reachable in dev namespace.
- App-to-app communication working.

## Phase 4: Centralized Infra Repo (Week 4)

### Outcome
One source of truth for platform declarations.

### Tasks

1. Create a new repo: infra-k8s.
2. Add structure:
   - apps/
   - base/
   - overlays/dev
   - overlays/stage
3. Choose one packaging approach:
   - Kustomize (simpler start), or
   - Helm (industry common)
4. Move deployment declarations here.

### Deliverables

- infra-k8s controls all environment manifests.
- One command deploys dev overlay.

## Phase 5: Reliability Basics (Week 5)

### Outcome
Safer rollouts and predictable behavior under load.

### Tasks

1. Add readiness/liveness probes for all services.
2. Add resources requests/limits.
3. Add HorizontalPodAutoscaler for one stateless app.
4. Practice rollout restart and rollback.

### Deliverables

- Documented rollback workflow.
- At least one autoscaled service.

## Phase 6: Security Basics (Week 6)

### Outcome
Cluster has baseline access and network controls.

### Tasks

1. Create service accounts per app.
2. Add RBAC roles with least privilege.
3. Add NetworkPolicies (deny by default + explicit allow).
4. Run containers as non-root where possible.

### Deliverables

- Working RBAC for app operations.
- Network policy enforcing app boundaries.

## Phase 7: Observability (Week 7)

### Outcome
You can detect and troubleshoot production issues.

### Tasks

1. Install Prometheus + Grafana.
2. Add dashboards for CPU, memory, restarts, latency.
3. Add centralized logs (Loki or cloud logging).
4. Add 2-3 alerts (high error rate, pod crash loop, high latency).

### Deliverables

- One dashboard per critical app.
- At least one tested alert.

## Phase 8: GitOps + CI/CD (Week 8)

### Outcome
Declarative auto-deploy from Git.

### Tasks

1. Install Argo CD.
2. Connect Argo CD to infra-k8s repo.
3. In each app repo, build and push images on commit.
4. Update image tags in infra-k8s via pull request.
5. Let Argo CD sync changes to cluster.

### Deliverables

- Commit to infra repo triggers deployment.
- Rollback possible by Git revert.

## Phase 9: Google-Relevant Track with GKE (Week 9)

### Outcome
Run same workloads on GKE.

### Tasks

1. Create a GKE Autopilot cluster.
2. Apply same manifests/charts.
3. Configure ingress and managed certificates.
4. Use Workload Identity for app permissions.

### Deliverables

- Same services running in GKE.
- Notes on differences: k3s vs GKE.

## Phase 10: Capstone (Week 10)

### Outcome
Portfolio-grade platform project.

### Tasks

1. Perform failure drill (break one service and recover).
2. Perform zero-downtime deployment test.
3. Write runbooks:
   - Deploy
   - Rollback
   - Incident response
4. Record architecture diagram and final checklist.

### Deliverables

- End-to-end demo environment.
- Documentation ready for interview discussion.

## Weekly Routine

Use this rhythm every week:

1. Build one feature in cluster (60%).
2. Break and fix something intentionally (20%).
3. Write short notes and runbook updates (20%).

## Minimum Commands to Master

- kubectl get pods -A
- kubectl describe pod <name> -n <namespace>
- kubectl logs <pod> -n <namespace>
- kubectl apply -f <file>
- kubectl rollout status deployment/<name> -n <namespace>
- kubectl rollout undo deployment/<name> -n <namespace>
- kubectl port-forward svc/<name> 8080:80 -n <namespace>

## Definition of Done

You are ready for real team workflows when you can:

1. Deploy 3+ services with ingress and TLS.
2. Roll out and roll back safely.
3. Diagnose a failing deployment in under 30 minutes.
4. Explain resource limits, probes, and autoscaling choices.
5. Run same stack locally and on GKE.

## Next Action (Today)

1. Install k3s.
2. Pick model-api as first app.
3. Create Deployment + Service YAML.
4. Verify app is reachable.
5. Write one short note: what failed, how you fixed it.
