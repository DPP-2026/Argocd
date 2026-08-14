# Step 3 — Install Argo Rollouts

Argo Rollouts extends Kubernetes with progressive delivery strategies (canary, blue/green). ArgoCD can deploy `Rollout` resources the same way it deploys `Deployment` resources.

## Install the Controller

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

Verify:

```bash
kubectl get pods -n argo-rollouts
kubectl get crd rollouts.argoproj.io
```

## Install kubectl Plugin (optional but recommended)

**macOS:**

```bash
brew install argoproj/tap/kubectl-argo-rollouts
```

**Linux:**

```bash
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

Test:

```bash
kubectl argo rollouts version
```

## What Argo Rollouts Adds

| Standard Deployment | Argo Rollout |
|---------------------|--------------|
| Rolling update only | Canary, blue/green, experiments |
| All-or-nothing pod replacement | Weighted traffic steps with pauses |
| `kubectl rollout status` | `kubectl argo rollouts get rollout` |

The guestbook rollout manifest lives at `manifests/guestbook-rollout/rollout.yaml`. Deploy and operate it after [Applications & Projects](04-applications-and-projects.md) — see [Argo Rollouts basics & notes](06-argo-rollouts-basics-and-notes.md).

## Next Step

→ [Applications & Projects](04-applications-and-projects.md)
