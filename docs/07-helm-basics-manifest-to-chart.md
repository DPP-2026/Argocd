# Step 7 — Helm Basics: Manifest to Chart

**Prerequisite for:** [Helm deployment via ArgoCD](helm-deployment-argocd.md)

This guide shows how plain Kubernetes YAML in this repo maps to a Helm chart. After this step you can deploy the same guestbook app with one chart and environment-specific values instead of three duplicate manifest folders.

## Why Helm?

Today the guestbook app lives in three nearly identical folders:

| Folder | Replicas | Environment label |
|--------|----------|---------------------|
| `manifests/guestbook/dev/` | 2 | `dev` |
| `manifests/guestbook/staging/` | 2 | `staging` |
| `manifests/guestbook/prod/` | 3 | `prod` |

Image, ports, and env vars are the same everywhere. Helm replaces copy-paste with **one template** and **values files** that only override what differs per environment.

## Mental Model

```text
Plain YAML:  deployment.yaml  =  final manifest (everything baked in)

Helm:        values.yaml       =  the parts you change per environment
             templates/*.yaml  =  the skeleton with {{ placeholders }}
             helm template       =  produces the same plain YAML kubectl would apply
```

## Install Helm

**macOS:**

```bash
brew install helm
```

**Linux:**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify:

```bash
helm version
```

## Side-by-Side: Plain YAML vs Helm

### Plain Kubernetes manifest (what you have today)

From `manifests/guestbook/dev/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-ui
  labels:
    app: guestbook-ui
    environment: dev          # hardcoded per env folder
spec:
  replicas: 2                 # hardcoded (dev=2, prod=3)
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
        environment: dev
    spec:
      containers:
        - name: guestbook-ui
          image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
          env:
            - name: GET_HOSTS_FROM
              value: dns
          ports:
            - containerPort: 80
```

### Helm chart layout

Create this structure (or use `helm create charts/guestbook` and replace the generated files):

```text
charts/guestbook/
├── Chart.yaml
├── values.yaml              # defaults (dev)
├── values-staging.yaml      # staging overrides
├── values-prod.yaml         # prod overrides
└── templates/
    ├── deployment.yaml      # one template for all envs
    └── service.yaml
```

### `Chart.yaml`

```yaml
apiVersion: v2
name: guestbook
description: Guestbook UI Helm chart
type: application
version: 0.1.0
appVersion: "v5"
```

### `values.yaml` — defaults (dev)

Everything that was **hardcoded** in the plain YAML moves here:

```yaml
replicaCount: 2

image:
  repository: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend
  tag: v5

environment: dev

revisionHistoryLimit: 3

service:
  type: ClusterIP
  port: 80

env:
  GET_HOSTS_FROM: dns
```

### `values-staging.yaml` — staging overrides

```yaml
environment: staging
```

### `values-prod.yaml` — prod overrides

```yaml
replicaCount: 3
environment: prod
```

Helm merges override files **on top of** `values.yaml`. Image, port, and env stay from defaults unless you change them.

### `templates/deployment.yaml` — Helm template

Same Deployment, but literals become `{{ .Values.* }}`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-ui
  labels:
    app: guestbook-ui
    environment: {{ .Values.environment }}
spec:
  replicas: {{ .Values.replicaCount }}
  revisionHistoryLimit: {{ .Values.revisionHistoryLimit }}
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
        environment: {{ .Values.environment }}
    spec:
      containers:
        - name: guestbook-ui
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          env:
            - name: GET_HOSTS_FROM
              value: {{ .Values.env.GET_HOSTS_FROM | quote }}
          ports:
            - containerPort: 80
```

### `templates/service.yaml`

**Plain YAML** (`manifests/guestbook/dev/service.yaml`):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui
  labels:
    app: guestbook-ui
    environment: dev
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: guestbook-ui
```

**Helm template:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui
  labels:
    app: guestbook-ui
    environment: {{ .Values.environment }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
  selector:
    app: guestbook-ui
```

## Field Mapping Reference

| Plain YAML (hardcoded) | Helm values key | Helm template |
|------------------------|-----------------|---------------|
| `replicas: 2` | `replicaCount: 2` | `{{ .Values.replicaCount }}` |
| `environment: dev` | `environment: dev` | `{{ .Values.environment }}` |
| `image: .../gb-frontend:v5` | `image.repository` + `image.tag` | `"{{ .Values.image.repository }}:{{ .Values.image.tag }}"` |
| `revisionHistoryLimit: 3` | `revisionHistoryLimit: 3` | `{{ .Values.revisionHistoryLimit }}` |
| `GET_HOSTS_FROM: dns` | `env.GET_HOSTS_FROM: dns` | `{{ .Values.env.GET_HOSTS_FROM \| quote }}` |
| `type: ClusterIP` | `service.type: ClusterIP` | `{{ .Values.service.type }}` |

## Lab Exercise — Render and Compare

Scaffold the chart (optional starting point):

```bash
helm create charts/guestbook
```

Replace the generated `templates/` and `values.yaml` with the examples above, then render without installing to a cluster:

```bash
# Dev — should match manifests/guestbook/dev/
helm template guestbook charts/guestbook

# Staging
helm template guestbook charts/guestbook -f charts/guestbook/values-staging.yaml

# Prod — should match manifests/guestbook/prod/
helm template guestbook charts/guestbook -f charts/guestbook/values-prod.yaml
```

Validate the chart:

```bash
helm lint charts/guestbook
```

Compare rendered dev output to the plain manifest:

```bash
helm template guestbook charts/guestbook > /tmp/guestbook-helm-dev.yaml
diff manifests/guestbook/dev/deployment.yaml <(yq '. | select(.kind == "Deployment")' /tmp/guestbook-helm-dev.yaml)
```

> `yq` is optional — the goal is to confirm replicas, image, and labels match after rendering.

### Expected rendered output (dev excerpt)

After `helm template`, the Deployment section should look like your plain YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook-ui
  labels:
    app: guestbook-ui
    environment: dev
spec:
  replicas: 2
  revisionHistoryLimit: 3
  ...
  containers:
    - name: guestbook-ui
      image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
```

For prod, only `replicas` and `environment` labels change:

```yaml
  labels:
    environment: prod
spec:
  replicas: 3
```

## Install to Cluster (optional)

To test outside ArgoCD:

```bash
helm install guestbook-dev charts/guestbook -n dev --create-namespace
helm install guestbook-prod charts/guestbook -n prod --create-namespace \
  -f charts/guestbook/values-prod.yaml
```

Uninstall when done:

```bash
helm uninstall guestbook-dev -n dev
helm uninstall guestbook-prod -n prod
```

## How This Connects to ArgoCD

**Today** — Application points at a folder of plain YAML:

```yaml
source:
  repoURL: https://github.com/DPP-2026/Argocd.git
  path: manifests/guestbook/dev
```

**With Helm** — Application points at the chart and selects values:

```yaml
source:
  repoURL: https://github.com/DPP-2026/Argocd.git
  path: charts/guestbook
  helm:
    valueFiles:
      - values-prod.yaml
```

ArgoCD runs the same `helm template` step internally and applies the rendered manifests.

## What You Learned

- Chart layout: `Chart.yaml`, `values.yaml`, `templates/`
- Hardcoded manifest fields become `.Values` entries
- One template replaces duplicate dev/staging/prod YAML folders
- `helm template` previews the exact YAML that would be applied
- This is the prerequisite before [Helm deployment via ArgoCD](helm-deployment-argocd.md)

## Next Step

→ [Helm deployment via ArgoCD](helm-deployment-argocd.md)
