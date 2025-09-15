# project-poc
# Three-tier Microservice Deployment on Kubernetes

**Stack**: Azure DevOps (ADO) + Docker Hub + ArgoCD + Istio + Flagger + Kubernetes

**Goal**: Deploy a three-tier microservice application (Frontend, Backend, Database) to Kubernetes with GitOps (ArgoCD), progressive delivery (Flagger + Istio canaries), CI/CD (Azure DevOps), image registry (Docker Hub), secrets & keys (Azure Key Vault / ADO variable groups), monitoring (Prometheus + Grafana), and production-grade concerns (TLS/mTLS, image signing, rollbacks, observability).

---

## Table of contents

1. Overview & architecture
2. Git repository layout
3. Prerequisites (cloud, tooling, accounts)
4. High-level steps
5. Detailed implementation (repos, manifests, Helm charts)

   * Kubernetes namespaces & RBAC
   * Helm chart + values for each service
   * Deployments: Deployments/StatefulSets, Services, Ingress (Istio Gateway & VirtualService)
   * Istio configuration (Gateway, VirtualService, DestinationRule, PeerAuthentication)
   * Flagger Canary CR and metrics (Prometheus)
   * ArgoCD Application manifests (App-of-Apps pattern)
   * Azure DevOps pipelines (build + release) YAMLs
   * Dockerfile examples
   * MySQL (stateful) storage & backup strategy
   * Secrets management with Azure Key Vault & AAD Pod Identity (or CSI driver)
   * Image signing suggestion (cosign) and enforce with admission control
6. Observability & monitoring
7. Canary strategy & SLOs
8. Security & compliance
9. Testing, validation, troubleshooting, rollback
10. Deliverables (files to commit)
11. Checklist & runbook
12. Appendices: sample manifest snippets

---

## 1. Overview & architecture

* **Frontend**: React (static build served by nginx) — stateless Deployment
* **Backend**: .NET / Node.js microservice — stateless Deployment
* **Database**: MySQL — StatefulSet + PVC

Traffic flow:

* External traffic -> Istio Ingress Gateway -> VirtualService -> Backend & Frontend
* Istio handles routing, mTLS (inside mesh), telemetry
* Flagger uses Istio VirtualService and DestinationRule to perform canary analysis using Prometheus metrics and tests
* ArgoCD watches the `infrastructure` Git repo and applies manifests to k8s
* Azure DevOps builds container images, pushes to Docker Hub, updates Git manifests (or Helm Chart version / values), and triggers ArgoCD sync

## 2. Git repository layout (recommended)

We will use two Git repos (or three):

1. `app-src` — application source code (frontend, backend)

   * `frontend/` (Dockerfile, app source)
   * `backend/` (Dockerfile, app source)

2. `k8s-manifests` (GitOps repo watched by ArgoCD)

   * `charts/` or `helm/` (helm charts for frontend, backend, mysql)
   * `envs/`

     * `dev/` (values.yaml, kustomize overlays if used)
     * `staging/`
     * `prod/`
   * `argo-apps/` (App-of-Apps manifests)
   * `istio/` (istio resources shared across apps)
   * `prometheus/`, `grafana/`

3. Optional `infra` repo for cluster provisioning (Terraform / ARM / bicep)

## 3. Prerequisites

* Kubernetes cluster (AKS recommended) with control plane and worker pools.
* kubectl, helm, istioctl installed locally.
* ArgoCD installed in cluster (namespace `argocd`).
* Istio installed (Istio operator or Helm).
* Flagger installed and able to talk to Prometheus and Istio.
* Prometheus + Grafana deployed (Flagger needs Prometheus).
* Azure DevOps project and a Service Connection (Kubernetes / ACR if used). For Docker Hub, create Docker Hub connection in ADO service connections.
* Docker Hub account and a repository for images.
* Azure Key Vault (if using) and an ADO service principal with access.

## 4. High-level steps

1. Provision Kubernetes cluster and install base components (Istio, ArgoCD, Prometheus, Flagger).
2. Create Git repos and structure (app-src, k8s-manifests).
3. Implement Helm charts for each tier.
4. Add Istio Gateway/VirtualService and Flagger Canary CRs to `k8s-manifests`.
5. Create Azure DevOps pipelines:

   * CI pipeline: build images for frontend/backend, push to Docker Hub, run SCA/SAST, sign images.
   * CD pipeline: update `k8s-manifests` (image tags or Helm chart values) and create a Git commit to trigger ArgoCD.
6. Configure ArgoCD to sync repo and set App-of-Apps to manage environment apps.
7. Configure Flagger policies & analysis (metrics, webhooks, hooks for tests).
8. Validate canary rollout in staging -> promote to prod.
9. Add monitoring alerts, SLOs, and runbook for rollbacks.

## 5. Detailed implementation

> NOTE: The files listed in the Deliverables (Section 10) are the ones to commit in the repos. Here are examples and templates.

### 5.1 Namespaces & RBAC

* Namespaces: `argocd`, `istio-system`, `monitoring`, `dev`, `staging`, `prod`.

Create `namespaces.yaml` (example):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```

RBAC for ArgoCD (if using cluster-scoped): use the ArgoCD recommended RBAC config. For Flagger, ensure it has permissions to modify VirtualService and DestinationRule.

### 5.2 Helm chart structure (app chart template)

`charts/frontend/`:

* `Chart.yaml`
* `values.yaml`
* `templates/deployment.yaml`
* `templates/service.yaml`
* `templates/ingress.yaml` (for istio, create Gateway/VirtualService YAMLs separately)

`values.yaml` key fields:

```yaml
replicaCount: 2
image:
  repository: yourdockerhub/frontend
  tag: "0.1.0"
service:
  port: 80
```

### 5.3 Deployments & Services (example deployment for backend)

`templates/deployment.yaml` (helm):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "fullname" . }}
  labels:
    app: {{ include "name" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "name" . }}
  template:
    metadata:
      labels:
        app: {{ include "name" . }}
    spec:
      containers:
      - name: {{ include "name" . }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 30
```

### 5.4 Istio Gateway & VirtualService

Create a shared Gateway for domain `example.com` and VirtualService routing to services.

`istio/gateway.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```

`istio/frontend-vs.yaml` (VirtualService):

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: frontend
  namespace: prod
spec:
  hosts:
  - "frontend.example.com"
  gateways:
  - istio-system/my-gateway
  http:
  - route:
    - destination:
        host: frontend.prod.svc.cluster.local
        port:
          number: 80
```

Include `DestinationRule` with subsets used by Flagger for canary routing.

### 5.5 Flagger Canary CR (example)

Flagger uses Canary custom resources and will modify VirtualService weights during analysis.

`canaries/backend-canary.yaml`:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: backend
  namespace: prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  progressDeadlineSeconds: 600
  service:
    port: 80
    portName: http
  analysis:
    interval: 1m
    threshold: 10
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      interval: 1m
      provider:
        type: prometheus
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: "(sum(rate(istio_requests_total{destination_service=\"backend.prod.svc.cluster.local\",response_code!~\"5.*\"}[5m])) / sum(rate(istio_requests_total{destination_service=\"backend.prod.svc.cluster.local\"}[5m])))*100"
    webhooks:
    - name: acceptance-test
      url: http://flagger-webhook.prod.svc.cluster.local/acceptance
      timeout: 5s
```

This example uses Prometheus queries and a webhook for functional tests.

### 5.6 ArgoCD Applications (App-of-Apps)

Create an `argo-apps/parent-app.yaml` which points to `envs/dev` and `envs/prod` child apps.

`argo-apps/frontend-app.yaml` example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://your-git/k8s-manifests'
    targetRevision: HEAD
    path: envs/dev/frontend
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Use app-of-apps: one parent Application referencing a folder containing child Application YAMLs.

### 5.7 Azure DevOps Pipelines

We will create two pipeline YAMLs in `app-src` repo:

#### CI: `.ado/pipelines/ci.yml`

* Steps:

  1. Checkout
  2. Build frontend and backend Docker images
  3. Run unit tests, SAST (SonarQube), SCA (OWASP dependency-check or aquasecurity), container scan (Trivy)
  4. Sign images with `cosign` (optional)
  5. Push images to Docker Hub
  6. Publish image tags as pipeline artifacts (or update manifests repo)

Sample pipeline snippet (simplified):

```yaml
trigger:
  branches: [ main ]

pool:
  vmImage: 'ubuntu-latest'

variables:
  DOCKERHUB_REPO: yourdockerhub

steps:
- task: Docker@2
  displayName: Build and push backend
  inputs:
    command: buildAndPush
    repository: $(DOCKERHUB_REPO)/backend
    dockerfile: backend/Dockerfile
    tags: $(Build.BuildId)
    containerRegistry: DockerHubServiceConnection

- task: Docker@2
  displayName: Build and push frontend
  inputs:
    command: buildAndPush
    repository: $(DOCKERHUB_REPO)/frontend
    dockerfile: frontend/Dockerfile
    tags: $(Build.BuildId)
    containerRegistry: DockerHubServiceConnection

# Optionally run security scans and cosign
```

#### CD: `.ado/pipelines/cd.yml`

* Steps:

  1. Checkout `k8s-manifests` repo (use personal access token or service connection)
  2. Update Helm `values.yaml` with new image tags (or update kustomize image tags)
  3. Commit & push changes back to `k8s-manifests` repository
  4. Optionally call ArgoCD API to sync (or rely on ArgoCD auto-sync)

Snippet (pseudo steps):

```yaml
trigger: none
parameters:
- name: imageTag
  type: string

steps:
- checkout: self
- script: |
    git config user.email "ado-bot@yourdomain.com"
    git config user.name "ADO CI"
    yq eval -i ".image.tag = \"$(imageTag)\"" envs/prod/backend/values.yaml
    git add .
    git commit -m "chore: update backend image to $(imageTag)"
    git push origin HEAD:main
  displayName: Update manifests repo
# Optionally call ArgoCD API to start sync if not automated
```

Important: Use a service principal or PAT with minimal permissions for committing updates.

### 5.8 Dockerfile examples

**Backend Dockerfile** (Node.js example):

```dockerfile
FROM node:18-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=build /app/dist ./
RUN npm ci --production
CMD ["node","index.js"]
```

**Frontend Dockerfile** (React served by nginx):

```dockerfile
FROM node:18-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:stable-alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx","-g","daemon off;"]
```

### 5.9 Database (MySQL) StatefulSet & PVC

Use a StatefulSet or managed DB (recommended: managed Azure Database for MySQL). If using in-cluster MySQL:

`mysql/statefulset.yaml` with a `StorageClass` like `managed-nfs` or cloud block storage.

Also include a `Backup` strategy: Velero or scheduled mysqldump to object storage.

### 5.10 Secrets & Key Management

Options:

* **Azure Key Vault + AAD Pod Identity + Key Vault CSI driver**: mount secrets as files or sync to k8s secrets.
* **ArgoCD Vault Plugin**: fetch secrets at sync time.
* **SealedSecrets / ExternalSecrets**: alternatives.

Store Docker Hub credentials in ADO pipeline library (secure variable group) and in DockerHub service connection.

### 5.11 Image Signing & Policy

* Use `cosign` to sign images in CI.
* Deploy an admission controller (e.g., `sigstore` or `cosign` admission webhook) to require signature verification.

## 6. Observability & monitoring

* Deploy Prometheus (monitoring namespace) for metrics scraping.
* Configure Istio telemetry with Prometheus.
* Flagger relies on Prometheus to compute error rates and latency.
* Grafana dashboards: Istio dashboard, application dashboard, Flagger dashboard.
* Configure alertmanager for key alerts (high error rate, high latency, pod restarts).

## 7. Canary strategy & SLOs

* Start with short intervals and small steps: stepWeight 5-10%, maxWeight 50%.
* Analysis metrics: success rate (>99%), latency (p95 below X ms), error rate < 1%.
* Define promotion rules: After x minutes at maxWeight with stable metrics, promote to 100%.
* Define automated rollbacks: Flagger will rollback on threshold breach.

## 8. Security & compliance

* Network policies for namespace isolation.
* Istio mTLS inside mesh.
* Pod security standards (use PSP alternatives: PodSecurity admission or OPA/Gatekeeper).
* Image scanning in CI and admission-time enforcement.
* Least privilege for ADO service accounts and GitOps write tokens.

## 9. Testing, validation & troubleshooting

* Local dev: use kind/minikube with Istio & ArgoCD for smoke testing.
* Use Flagger's `canary-validate` scripts to run functional tests as webhooks.
* For troubleshooting: check ArgoCD `Application` status, `kubectl describe canary`, `kubectl -n prod get virtualservice`, Flagger logs, Istio ingressgateway logs.

## 10. Deliverables (files to commit)

**app-src**:

* `frontend/` (source + Dockerfile)
* `backend/` (source + Dockerfile)
* `.ado/pipelines/ci.yml`
* `.ado/pipelines/cd.yml`

**k8s-manifests**:

* `charts/frontend/` (Helm chart)
* `charts/backend/` (Helm chart)
* `charts/mysql/` (StatefulSet)
* `envs/dev/*`, `envs/staging/*`, `envs/prod/*` (values and overlays)
* `istio/gateway.yaml`, `istio/destinationrule.yaml`, `istio/virtualservice-*.yaml`
* `canaries/backend-canary.yaml`, `canaries/frontend-canary.yaml`
* `argo-apps/` (App-of-Apps parent and child apps)
* `monitoring/prometheus-flagger.yaml`, `monitoring/grafana-dashboard-flagger.yaml`
* `namespaces.yaml`, `rbac/` files

## 11. Checklist & Runbook (short)

* [ ] Cluster provisioned and accessible
* [ ] Istio installed and sidecar injection configured per namespace
* [ ] ArgoCD installed and bootstrapped
* [ ] Prometheus + Grafana installed
* [ ] Flagger installed and connected to Istio + Prometheus
* [ ] Azure DevOps pipelines created and secret/service connections configured
* [ ] Docker images build & push successful
* [ ] ArgoCD Application syncs and pods reach `Ready`
* [ ] Canary runs and observed by Flagger
* [ ] Alerting configured

Runbook actions (if canary fails):

1. Check Flagger logs: `kubectl -n prod logs deployment/flagger`
2. Describe canary: `kubectl -n prod describe canary backend`
3. Check Prometheus queries for metric anomalies
4. Rollback: Flagger should auto-rollback; otherwise revert manifest change in git and ArgoCD will reconcile.

## 12. Appendix: key manifest snippets

(See Deliverables folder for full files). A couple more quick templates:

**ArgoCD app-of-apps parent**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: apps-root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://your-git/k8s-manifests'
    path: argo-apps
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Flagger Canary summary**

* `targetRef` -> deployment
* `service` -> service port
* `analysis` -> metrics + webhooks

---

## Next steps I can do for you (pick any)

* Generate the full set of ready-to-commit files (Helm charts, ArgoCD app files, ADO YAMLs) and place them in a zip for download.
* Create example CI/CD Azure DevOps YAMLs with cosign integration and security scanning integrated.
* Produce a step-by-step runbook for cluster bootstrap (commands for AKS + Istio + ArgoCD + Flagger + Prometheus).

Tell me which deliverable you want first and I will generate the files.

---

*This document is a comprehensive plan and template reference to implement a three-tier microservice deployment with GitOps and progressive delivery.*
