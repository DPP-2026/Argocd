# Step 6 — Argo Rollouts Basics & Notes

**Prerequisite:** [Install Argo Rollouts](03-install-argo-rollouts.md) (Step 3)

**Related advanced guides:** [Canary deployment](canary-deployment.md) · [Blue-green deployment](blue-green-deployment.md)

Argo Rollouts is a Kubernetes controller that replaces the standard `Deployment` rolling update with **progressive delivery** — canary, blue/green, and experiments. Argo CD deploys `Rollout` resources the same way it deploys `Deployment` resources.

## Deployment vs Rollout

| | Standard `Deployment` | Argo `Rollout` |
|---|----------------------|----------------|
| **Kind** | `apps/v1 Deployment` | `argoproj.io/v1alpha1 Rollout` |
| **Update strategy** | Rolling update only | Canary, blue/green, experiments |
| **Traffic control** | All pods updated together | Weighted steps with pauses |
| **Rollback** | `kubectl rollout undo` | `kubectl argo rollouts undo` |
| **Status** | `kubectl rollout status` | `kubectl argo rollouts get rollout` |
| **Argo CD** | Syncs normally | Syncs normally — same Application pattern |

The pod template (`spec.template`) looks almost identical. The difference is **`spec.strategy`**.

## Mental Model

```text
Git change (new image)
    → Argo CD syncs Rollout manifest
    → Rollout controller creates canary ReplicaSet
    → Steps run: 25% → pause → 50% → pause → 100%
    → Stable version fully replaced when complete
```

With a standard Deployment, Kubernetes replaces pods incrementally with no built-in pause or traffic weight control.

## Side-by-Side: Deployment vs Rollout

### Standard Deployment (this repo)

From `manifests/guestbook/dev/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-ui
spec:
  replicas: 2
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
    spec:
      containers:
        - name: guestbook-ui
          image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
          ports:
            - containerPort: 80
  # Deployment uses rollingUpdate by default — no steps, no pauses
```

### Argo Rollout (this repo)

From `manifests/guestbook-rollout/rollout.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: guestbook-ui
spec:
  replicas: 3
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
    spec:
      containers:
        - name: guestbook-ui
          image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
          ports:
            - containerPort: 80
  strategy:
    canary:
      stableService: guestbook-ui           # traffic to stable pods
      canaryService: guestbook-ui-canary    # traffic to canary pods
      steps:
        - setWeight: 25
        - pause: { duration: 30s }
        - setWeight: 50
        - pause: { duration: 30s }
        - setWeight: 100
```

**What changed:** `kind` is `Rollout`, and `strategy.canary` defines how the new version is promoted.

## Key Concepts

### Stable vs Canary

| Term | Meaning |
|------|---------|
| **Stable** | Current production version (old ReplicaSet) |
| **Canary** | New version being tested (new ReplicaSet) |
| **stableService** | Service selector pointing at stable pods |
| **canaryService** | Service selector pointing at canary pods |

This repo defines both services in `manifests/guestbook-rollout/service.yaml`:

```yaml
# Stable service
metadata:
  name: guestbook-ui
---
# Canary service
metadata:
  name: guestbook-ui-canary
```

Both select `app: guestbook-ui`. The Rollout controller updates pod labels so each service routes to the correct ReplicaSet during the rollout.

### Canary Steps

| Step | What it does |
|------|--------------|
| `setWeight: 25` | Send 25% of traffic to canary pods |
| `pause: { duration: 30s }` | Wait 30 seconds before next step |
| `pause: {}` | Wait indefinitely until manual promote |
| `setWeight: 100` | All traffic on new version — rollout complete |

### Blue/Green (alternative strategy)

Instead of gradual weights, blue/green runs the new version alongside the old, then switches traffic in one step:

```yaml
strategy:
  blueGreen:
    activeService: guestbook-ui        # live traffic
    previewService: guestbook-ui-canary  # preview new version
    autoPromotionEnabled: false        # require manual promote
```

See [blue-green deployment](blue-green-deployment.md) for a full ingress-based lab.

## Deploy via Argo CD

Application manifest: `applications/guestbook-rollout.yaml`

```yaml
spec:
  project: guestbook
  source:
    path: manifests/guestbook-rollout
  destination:
    namespace: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=false
```

Note: this Application has **manual sync** (no `automated` block). You trigger rollouts explicitly with `argocd app sync guestbook-rollout`.

Deploy:

```bash
kubectl apply -f applications/guestbook-rollout.yaml
argocd app sync guestbook-rollout
```

Verify:

```bash
kubectl argo rollouts get rollout guestbook-ui
kubectl get pods -l app=guestbook-ui
```

## Lab Exercise — Watch a Canary Rollout

### Step 1: Deploy the Rollout

```bash
kubectl apply -f applications/guestbook-rollout.yaml
argocd app sync guestbook-rollout
```

### Step 2: Watch status

```bash
kubectl argo rollouts get rollout guestbook-ui --watch
```

Expected phases: `Progressing` → pauses at 25% and 50% → `Healthy` at 100%.

### Step 3: Trigger a new rollout via Git

Edit `manifests/guestbook-rollout/rollout.yaml` — change replicas or the image field, commit, and push:

```bash
# Example: scale replicas 3 → 4
git add manifests/guestbook-rollout/rollout.yaml
git commit -m "Trigger guestbook rollout scale"
git push origin main
```

Sync and watch:

```bash
argocd app sync guestbook-rollout
kubectl argo rollouts get rollout guestbook-ui --watch
```

### Step 4: Port-forward to test (optional)

```bash
kubectl port-forward svc/guestbook-ui 8081:80
# Open http://localhost:8081
```

## Essential CLI Commands

```bash
# Status and history
kubectl argo rollouts get rollout guestbook-ui
kubectl argo rollouts history rollout guestbook-ui

# Control an in-progress rollout
kubectl argo rollouts promote guestbook-ui      # skip pause, continue
kubectl argo rollouts abort guestbook-ui         # cancel, roll back canary
kubectl argo rollouts retry guestbook-ui         # retry failed rollout
kubectl argo rollouts undo guestbook-ui          # rollback to previous version

# Dashboard (optional)
kubectl argo rollouts dashboard
```

## Strategy Comparison

| Strategy | Best for | Traffic switch | Rollback |
|----------|----------|----------------|----------|
| **Rolling update** (Deployment) | Simple apps, dev | Gradual, no control | `kubectl rollout undo` |
| **Canary** (Rollout) | Prod with metrics/observability | Weighted steps + pauses | `abort` or `undo` |
| **Blue/green** (Rollout) | Zero-downtime cutover | Instant switch | Switch back to preview |

## Important Notes

### 1. Rollout controller must be installed

The `Rollout` CRD and controller from Step 3 must be running:

```bash
kubectl get pods -n argo-rollouts
kubectl get crd rollouts.argoproj.io
```

Without the controller, Argo CD syncs the manifest but nothing reconciles the rollout.

### 2. Two services required for canary

Canary strategy needs **both** `stableService` and `canaryService` defined in the Rollout spec, plus matching Service manifests. Missing either causes the rollout to stall.

### 3. Traffic routing without Ingress

This lab's guestbook rollout uses **ReplicaSet-based canary** (pod weight steps only). For real traffic splitting by percentage, add a traffic router (NGINX Ingress, ALB, Istio, etc.) — see [canary deployment](canary-deployment.md).

### 4. Argo CD vs Rollout controller roles

| Component | Role |
|-----------|------|
| **Argo CD** | Keeps the Rollout manifest in sync with Git |
| **Rollout controller** | Executes canary/blue-green steps at runtime |

Changing Git triggers Argo CD sync → Rollout controller starts a new rollout.

### 5. Manual sync for rollouts in this lab

`guestbook-rollout` Application has no `automated` sync. Production teams often use manual or approval-based sync for rollouts so promotions are deliberate.

### 6. Image availability

Use `gb-frontend:v5` only — other guestbook tags are not published. See [05-sync-and-image-updates.md](05-sync-and-image-updates.md).

### 7. Pause indefinitely

`pause: {}` with no duration waits until you run:

```bash
kubectl argo rollouts promote guestbook-ui
```

Useful for manual verification or metric checks before full promotion.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Rollout stuck at `Paused` | Run `kubectl argo rollouts promote guestbook-ui` |
| `Rollout schema not found` | Install Rollouts controller (Step 3) |
| No canary pods created | Check `canaryService` and `stableService` names match Service manifests |
| Argo CD shows Synced but no rollout progress | Rollout controller may not be running — check `argo-rollouts` namespace |
| `ImagePullBackOff` | Use image tag `v5` only |

## What You Learned

- `Rollout` replaces `Deployment` when you need controlled progressive delivery
- Canary uses weighted steps and pauses; blue/green switches traffic in one cutover
- Argo CD deploys Rollouts the same way as Deployments — Git remains source of truth
- The Rollout controller (not Argo CD) executes promotion steps at runtime
- `kubectl argo rollouts` is the primary CLI for watching and controlling rollouts

## Next Step

→ [Helm basics: manifest to chart](07-helm-basics-manifest-to-chart.md)
