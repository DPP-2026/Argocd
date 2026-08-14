# Cleanup Resources

Follow this order **before** running `eksctl delete cluster` to avoid stuck drains and unevictable pod warnings.

## Step 1 — Delete ArgoCD Applications

```bash
argocd app delete guestbook-dev --cascade
argocd app delete guestbook-staging --cascade
argocd app delete guestbook-prod --cascade
argocd app delete guestbook-rollout --cascade
```

Or via kubectl:

```bash
kubectl delete -f applications/guestbook-dev.yaml --ignore-not-found
kubectl delete -f applications/guestbook-staging.yaml --ignore-not-found
kubectl delete -f applications/guestbook-prod.yaml --ignore-not-found
kubectl delete -f applications/guestbook-rollout.yaml --ignore-not-found
kubectl delete -f applications/project.yaml --ignore-not-found
```

## Step 2 — Delete Workload Namespaces

```bash
kubectl delete namespace dev --ignore-not-found
kubectl delete namespace staging --ignore-not-found
kubectl delete namespace prod --ignore-not-found
kubectl delete namespace argocd --ignore-not-found
kubectl delete namespace argo-rollouts --ignore-not-found
```

Wait until namespaces are gone:

```bash
kubectl get ns dev staging prod argocd argo-rollouts
```

## Step 3 — Delete Pod Disruption Budgets

Remove PDBs in `kube-system` so eksctl can drain nodes without hitting unevictable pod warnings (common with CoreDNS and metrics-server):

```bash
kubectl delete pdb -n kube-system coredns metrics-server --ignore-not-found
```

To remove any remaining PDBs across all namespaces:

```bash
kubectl get pdb -A --no-headers -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name' \
  | while read -r ns name; do kubectl delete pdb -n "$ns" "$name" --ignore-not-found; done
```

## Step 4 — Delete the EKS Cluster

```bash
eksctl delete cluster -f cluster/dev-cluster.yaml
```

Use the region from `cluster/dev-cluster.yaml` (currently `us-east-1`).

## If `eksctl delete` Hangs on "unevictable" Pods

eksctl drains nodes before delete. You may see:

```
1 pods are unevictable from node ip-xxx...
```

**Common cause:** CoreDNS PodDisruptionBudget — only one CoreDNS pod is running and the PDB blocks eviction.

### Option A — Skip node drain (fastest)

Cancel the stuck delete (`Ctrl+C`), then:

```bash
eksctl delete cluster -f cluster/dev-cluster.yaml --disable-nodegroup-eviction
```

### Option B — Remove PDBs, then delete normally

If you skipped Step 3, run the PDB deletion commands there, then retry:

```bash
eksctl delete cluster -f cluster/dev-cluster.yaml
```

### Option C — Find the blocking pod

```bash
kubectl get pods -A -o wide | grep <node-name-from-error>
kubectl get pdb -A
```

## Verify Cleanup

```bash
kubectl config get-contexts
aws eks list-clusters --region us-east-1
```

Replace `us-east-1` with your cluster region if different.
