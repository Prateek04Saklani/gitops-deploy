# GitOps Progressive Delivery

This repository manages progressive delivery deployments using **ArgoCD** and **Argo Rollouts** on Kubernetes. It covers two deployment strategies: **Blue-Green** and **Canary**.

## Repository Structure

```
.
├── apps/
│   ├── argocd-app-bluegreen.yaml   # ArgoCD Application for blue-green
│   └── argocd-app-canary.yaml      # ArgoCD Application for canary
├── blue-green/
│   ├── rollout.yaml                # Argo Rollouts Rollout resource
│   ├── service-active.yaml         # Service pointing to live (blue) pods
│   └── service-preview.yaml        # Service pointing to new (green) pods
└── canary/
    ├── rollout.yaml                # Argo Rollouts Rollout resource
    ├── analysistemplate.yaml       # Prometheus-based health analysis
    ├── ingress.yaml                # nginx Ingress for traffic routing
    ├── service-stable.yaml         # Service pointing to stable pods
    └── service-canary.yaml         # Service pointing to canary pods
```

## Prerequisites

- Kubernetes cluster
- ArgoCD installed in `argocd` namespace
- Argo Rollouts controller installed
- nginx Ingress controller
- Prometheus (in `monitoring` namespace, via kube-prometheus-stack)
- ECR pull secret named `ecr-pull-secret` in `default` namespace

---

## Blue-Green Deployment

### Overview

Blue-Green runs two identical environments simultaneously. The **active** (blue) environment serves live traffic while the **preview** (green) environment runs the new version. Promotion is manual, allowing full verification before switching traffic.

```
Users → app-active (blue)      app-preview (green) ← internal testing only
             │                          │
      revision:1 pods           revision:2 pods
      (current version)         (new version)
```

### Configuration

**Rollout** (`blue-green/rollout.yaml`)
- `replicas: 3` — 3 pods per environment (6 total during a rollout)
- `autoPromotionEnabled: false` — manual promotion required
- `activeService: app-active` — routes live traffic
- `previewService: app-preview` — routes preview traffic

**Services**
- `app-active` — always points to the live (promoted) ReplicaSet
- `app-preview` — always points to the new (pending) ReplicaSet

### Triggering a Rollout

Update `APP_VERSION` (or the image tag) in `blue-green/rollout.yaml` and merge to `main`. ArgoCD will sync and Argo Rollouts will create the green environment automatically.

```bash
# Watch rollout progress
kubectl argo rollouts get rollout app-bluegreen --watch
```

### Process Step by Step

1. **New version detected** — ArgoCD syncs the updated Rollout spec from `main`
2. **Green environment created** — Argo Rollouts creates a new ReplicaSet (revision:2); 3 new pods start up
3. **Preview traffic available** — `app-preview` service routes to the new pods; test internally before promoting
4. **Manual promotion** — After verifying the preview, promote to switch live traffic:

   ```bash
   kubectl argo rollouts promote app-bluegreen
   ```

5. **Traffic switches** — `app-active` service selector updates to point to revision:2 pods instantly (no downtime)
6. **Old environment scales down** — revision:1 pods are terminated

### Rollback

Before promotion, simply abort — revision:1 continues serving traffic unaffected:

```bash
kubectl argo rollouts abort app-bluegreen
```

After promotion, roll back by reverting the image/version in git and merging to `main`.

### Testing

**Test the live (active) environment:**
```bash
kubectl port-forward svc/app-active 8080:80
curl http://localhost:8080/
```

**Test the preview (green) environment before promoting:**
```bash
kubectl port-forward svc/app-preview 8081:80
curl http://localhost:8081/
```

---

## Canary Deployment

### Overview

Canary progressively shifts real user traffic from the stable version to the new version in weighted increments. An automated Prometheus-based health check gates promotion at each stage. If the canary fails the health check, the rollout aborts and traffic returns to stable.

```
Users → nginx Ingress (app.local)
              │
    ┌─────────┴──────────┐
 80% stable           20% canary    ← step 1
 50% stable           50% canary    ← step 4
  0% stable          100% canary    ← step 6 (promoted)
              │
       app-stable-svc         app-canary-svc
       (revision:1 pods)      (revision:2 pods)
```

Traffic splitting is handled by nginx at the ingress layer — independent of pod count.

### Configuration

**Rollout** (`canary/rollout.yaml`)
- `replicas: 5`
- Traffic routing via nginx Ingress (`app-ingress`)
- `canaryService: app-canary-svc` — receives canary traffic
- `stableService: app-stable-svc` — receives stable traffic

**Steps**
```
setWeight: 20  →  pause: 30s  →  analysis  →  setWeight: 50  →  pause: 30s  →  setWeight: 100
```

**AnalysisTemplate** (`canary/analysistemplate.yaml`)
- Provider: Prometheus at `http://prometheus-operated.monitoring.svc.cluster.local:9090`
- Metric: HTTP success rate (non-5xx / total requests) for `app="app-canary"`
- `count: 3` checks, `interval: 15s` (45 seconds total)
- `successCondition: result[0] >= 0.95` — requires ≥95% success rate
- `failureLimit: 2` — aborts if more than 2 checks fail
- Falls back to `vector(1)` (100% success) when no metrics exist

### Triggering a Rollout

Update `APP_VERSION` (or the image tag) in `canary/rollout.yaml` and merge to `main`.

```bash
# Watch rollout progress
kubectl argo rollouts get rollout app-canary --watch
```

### Process Step by Step

1. **New version detected** — ArgoCD syncs the updated Rollout spec from `main`
2. **Step 1 — setWeight: 20** — Argo Rollouts creates canary ReplicaSet (1 pod). nginx is configured to route 20% of traffic to `app-canary-svc`, 80% to `app-stable-svc`
3. **Step 2 — pause: 30s** — Rollout waits 30 seconds
4. **Step 3 — analysis** — An AnalysisRun is created. Prometheus is queried 3 times (every 15s):
   - Query calculates HTTP success rate for canary pods
   - Each result must be ≥ 0.95 (95%)
   - More than 2 failures → rollout **aborts** and reverts to stable
   - All checks pass → rollout **continues**
5. **Step 4 — setWeight: 50** — nginx weight updated to 50/50. Pod counts rebalance (~3 canary, ~3 stable)
6. **Step 5 — pause: 30s** — Rollout waits 30 seconds
7. **Step 6 — setWeight: 100** — Full promotion. All traffic shifts to canary pods. Stable ReplicaSet scales down to 0. Canary becomes the new stable

### Rollback

Automatic — if the AnalysisRun fails, Argo Rollouts aborts and reverts traffic to stable with no manual intervention needed.

Manual abort at any step:
```bash
kubectl argo rollouts abort app-canary
```

Retry after fixing the issue:
```bash
kubectl argo rollouts retry rollout app-canary
```

### Testing

**Test the stable version:**
```bash
kubectl port-forward svc/app-stable-svc 8080:80
curl http://localhost:8080/
```

**Test the canary version directly (during a rollout):**
```bash
kubectl port-forward svc/app-canary-svc 8081:80
curl http://localhost:8081/
```

**Watch traffic distribution during a rollout:**
```bash
kubectl port-forward svc/app-stable-svc 8080:80 &
while true; do curl -s http://localhost:8080/ && sleep 0.5; done
```

**Check AnalysisRun results:**
```bash
kubectl get analysisrun -l rollout=app-canary
```

---

## Comparison

| | Blue-Green | Canary |
|---|---|---|
| Traffic switch | Instant (full cutover) | Gradual (weighted) |
| Promotion | Manual | Automatic (after analysis) |
| Rollback | Manual (revert git) | Automatic (on analysis failure) |
| Resource usage | 2x pods during rollout | ~1.2x pods during rollout |
| Risk exposure | Zero until promotion | Progressive, limited blast radius |
| Best for | Stateful apps, compliance, staged verification | High-traffic services needing safe gradual rollout |

## ArgoCD Applications

Both strategies are managed as ArgoCD Applications in the `apps/` directory, watching the `main` branch with automated sync, pruning, and self-healing enabled.

```bash
# Check sync status
kubectl get application -n argocd

# Force sync
argocd app sync app-canary
argocd app sync app-bluegreen
```
