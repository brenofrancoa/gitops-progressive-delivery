# DevOps GitOps Lab Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a production-like GitOps lab on Kind with ArgoCD + Rollouts, Blue/Green + Canary deployments, monitoring, and HPA.

**Architecture:** Mono-repo with FastAPI app, Kustomize base/overlays, ArgoCD app-of-apps for platform components, Blue/Green via dual Deployments + Service selector, Canary via Rollout steps.

**Tech Stack:** Python/FastAPI, Docker, Kind, Kustomize, ArgoCD, Argo Rollouts, ingress-nginx, kube-prometheus-stack, metrics-server, GitHub Actions.

---

### Task 1: Scaffold repo layout and docs

**Files:**
- Create: `.gitignore`
- Create: `README.md`
- Create: `docs/diagrams/architecture.mmd`
- Create: `docs/screenshots/README.md`

**Step 1: Create scaffold files**

`.gitignore`
```
__pycache__/
*.pyc
.venv/
.env
dist/
build/
*.egg-info/
.pytest_cache/
.coverage
.DS_Store
.worktrees/
worktrees/
```

`README.md` (stub)
```
# DevOps GitOps Lab

## Architecture
(placeholder)

## Quickstart
(placeholder)

## GitOps Flow
(placeholder)

## Demo Scenarios
(placeholder)
```

`docs/diagrams/architecture.mmd`
```
flowchart LR
  Dev[Developer] -->|git push| Repo[GitHub Repo]
  Repo -->|sync| ArgoCD[ArgoCD]
  ArgoCD --> App[FastAPI App]
  ArgoCD --> Rollouts[Argo Rollouts]
  ArgoCD --> Ingress[Nginx Ingress]
  ArgoCD --> Mon[Prometheus/Grafana]
```

`docs/screenshots/README.md`
```
# Screenshots

Place screenshots here:
- argocd-apps.png
- argocd-rollouts.png
- grafana-dashboard.png
```

**Step 2: Sanity check**
Run: `git status`
Expected: new files listed as untracked

**Step 3: Commit**
Run:
```
git add .gitignore README.md docs/diagrams/architecture.mmd docs/screenshots/README.md
git commit -m "chore: scaffold repo docs"
```

---

### Task 2: FastAPI app + tests (TDD)

**Files:**
- Create: `app/requirements.txt`
- Create: `app/requirements-dev.txt`
- Create: `app/VERSION`
- Create: `app/tests/test_api.py`
- Create: `app/main.py`

**Step 1: Write failing tests**

`app/tests/test_api.py`
```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_healthz():
    resp = client.get("/healthz")
    assert resp.status_code == 200
    assert resp.json()["status"] == "ok"

def test_version():
    resp = client.get("/version")
    assert resp.status_code == 200
    assert "version" in resp.json()

def test_metrics():
    client.get("/")
    resp = client.get("/metrics")
    assert resp.status_code == 200
    assert "http_requests_total" in resp.text
```

**Step 2: Run tests to verify they fail**
Run: `pytest app/tests/test_api.py -q`
Expected: FAIL (module `main` not found)

**Step 3: Implement app + deps**

`app/requirements.txt`
```
fastapi==0.111.0
uvicorn[standard]==0.30.1
prometheus-client==0.20.0
```

`app/requirements-dev.txt`
```
pytest==8.2.2
httpx==0.27.0
```

`app/VERSION`
```
1.0.0
```

`app/main.py`
```python
import os
import time
from fastapi import FastAPI, Request, Response
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

REQUEST_COUNT = Counter(
    "http_requests_total", "Total HTTP requests", ["method", "path", "status"]
)
REQUEST_LATENCY = Histogram(
    "http_request_duration_seconds", "Request latency", ["path"]
)

app = FastAPI(title="lab-api")

@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    REQUEST_COUNT.labels(
        request.method, request.url.path, response.status_code
    ).inc()
    REQUEST_LATENCY.labels(request.url.path).observe(time.time() - start)
    return response

@app.get("/")
def root():
    return {"message": "lab-api"}

@app.get("/healthz")
def healthz():
    return {"status": "ok"}

@app.get("/version")
def version():
    return {"version": os.getenv("APP_VERSION", "dev")}

@app.get("/metrics")
def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)
```

**Step 4: Run tests to verify they pass**
Run:
```
pip install -r app/requirements.txt -r app/requirements-dev.txt
pytest app/tests/test_api.py -q
```
Expected: PASS

**Step 5: Commit**
Run:
```
git add app
git commit -m "feat: add fastapi app with metrics"
```

---

### Task 3: Containerization

**Files:**
- Create: `app/Dockerfile`
- Create: `app/.dockerignore`

**Step 1: Add Dockerfiles**

`app/Dockerfile`
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`app/.dockerignore`
```
__pycache__/
*.pyc
.venv/
tests/
.pytest_cache/
```

**Step 2: Build image**
Run: `docker build -t lab-api:local app`
Expected: build succeeds

**Step 3: Commit**
Run:
```
git add app/Dockerfile app/.dockerignore
git commit -m "chore: add docker build for app"
```

---

### Task 4: K8s base manifests (Blue/Green + Canary + HPA)

**Files:**
- Create: `k8s/base/kustomization.yaml`
- Create: `k8s/base/namespace.yaml`
- Create: `k8s/base/bluegreen.yaml`
- Create: `k8s/base/canary.yaml`
- Create: `k8s/base/ingress.yaml`
- Create: `k8s/base/hpa.yaml`

**Step 1: Create base manifests**

`k8s/base/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lab-devops
  labels:
    app.kubernetes.io/part-of: devops-lab
```

`k8s/base/bluegreen.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-blue
  namespace: lab-devops
  labels:
    app.kubernetes.io/name: lab-api
    color: blue
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: lab-api
      color: blue
  template:
    metadata:
      labels:
        app.kubernetes.io/name: lab-api
        color: blue
    spec:
      containers:
      - name: api
        image: docker.io/DOCKERHUB_USER/lab-api:v1.0.0
        ports:
        - containerPort: 8000
        envFrom:
        - configMapRef:
            name: app-config
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-green
  namespace: lab-devops
  labels:
    app.kubernetes.io/name: lab-api
    color: green
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: lab-api
      color: green
  template:
    metadata:
      labels:
        app.kubernetes.io/name: lab-api
        color: green
    spec:
      containers:
      - name: api
        image: docker.io/DOCKERHUB_USER/lab-api:v1.0.0
        ports:
        - containerPort: 8000
        envFrom:
        - configMapRef:
            name: app-config
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: api-bg
  namespace: lab-devops
spec:
  selector:
    app.kubernetes.io/name: lab-api
    color: blue # switch to green to flip traffic
  ports:
  - name: http
    port: 80
    targetPort: 8000
```

`k8s/base/canary.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-canary
  namespace: lab-devops
  labels:
    app.kubernetes.io/name: lab-api-canary
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: lab-api-canary
  template:
    metadata:
      labels:
        app.kubernetes.io/name: lab-api-canary
    spec:
      containers:
      - name: api
        image: docker.io/DOCKERHUB_USER/lab-api:v1.0.0
        ports:
        - containerPort: 8000
        envFrom:
        - configMapRef:
            name: app-config
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
  strategy:
    canary:
      stableService: api-canary-stable
      canaryService: api-canary-canary
      steps:
      - setWeight: 10
      - pause: {duration: "1m"}
      - setWeight: 30
      - pause: {duration: "1m"}
      - setWeight: 50
      - pause: {duration: "1m"}
      - setWeight: 100
---
apiVersion: v1
kind: Service
metadata:
  name: api-canary-stable
  namespace: lab-devops
spec:
  selector:
    app.kubernetes.io/name: lab-api-canary
  ports:
  - name: http
    port: 80
    targetPort: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: api-canary-canary
  namespace: lab-devops
spec:
  selector:
    app.kubernetes.io/name: lab-api-canary
  ports:
  - name: http
    port: 80
    targetPort: 8000
```

`k8s/base/ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-ingress
  namespace: lab-devops
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: lab.local
    http:
      paths:
      - path: /bg(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-bg
            port:
              number: 80
      - path: /canary(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-canary-stable
            port:
              number: 80
```

`k8s/base/hpa.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-canary-hpa
  namespace: lab-devops
spec:
  scaleTargetRef:
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    name: api-canary
  minReplicas: 2
  maxReplicas: 6
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
```

`k8s/base/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- namespace.yaml
- bluegreen.yaml
- canary.yaml
- ingress.yaml
- hpa.yaml
configMapGenerator:
- name: app-config
  literals:
  - APP_ENV=dev
  - APP_VERSION=v1.0.0
generatorOptions:
  disableNameSuffixHash: true
images:
- name: docker.io/DOCKERHUB_USER/lab-api
  newTag: v1.0.0
```

**Step 2: Render base**
Run: `kubectl kustomize k8s/base`
Expected: valid YAML output

**Step 3: Commit**
Run:
```
git add k8s/base
git commit -m "feat: add base k8s manifests for bluegreen and canary"
```

---

### Task 5: Kustomize overlays (dev/prod)

**Files:**
- Create: `k8s/overlays/dev/kustomization.yaml`
- Create: `k8s/overlays/dev/patch-service-blue.yaml`
- Create: `k8s/overlays/dev/patch-service-green.yaml`
- Create: `k8s/overlays/prod/kustomization.yaml`
- Create: `k8s/overlays/prod/patch-service-blue.yaml`
- Create: `k8s/overlays/prod/patch-service-green.yaml`

**Step 1: Add overlays**

`k8s/overlays/dev/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
- path: patch-service-blue.yaml
configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - APP_ENV=dev
  - APP_VERSION=v1.0.0
images:
- name: docker.io/DOCKERHUB_USER/lab-api
  newTag: v1.0.0
```

`k8s/overlays/dev/patch-service-blue.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-bg
  namespace: lab-devops
spec:
  selector:
    app.kubernetes.io/name: lab-api
    color: blue
```

`k8s/overlays/dev/patch-service-green.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-bg
  namespace: lab-devops
spec:
  selector:
    app.kubernetes.io/name: lab-api
    color: green
```

`k8s/overlays/prod/kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
patches:
- path: patch-service-blue.yaml
configMapGenerator:
- name: app-config
  behavior: merge
  literals:
  - APP_ENV=prod
  - APP_VERSION=v1.0.0
images:
- name: docker.io/DOCKERHUB_USER/lab-api
  newTag: v1.0.0
```

`k8s/overlays/prod/patch-service-blue.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-bg
  namespace: lab-devops
spec:
  selector:
    app.kubernetes.io/name: lab-api
    color: blue
```

`k8s/overlays/prod/patch-service-green.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-bg
  namespace: lab-devops
spec:
  selector:
    app.kubernetes.io/name: lab-api
    color: green
```

**Step 2: Render overlays**
Run:
```
kubectl kustomize k8s/overlays/dev
kubectl kustomize k8s/overlays/prod
```
Expected: valid YAML output

**Step 3: Commit**
Run:
```
git add k8s/overlays
git commit -m "feat: add kustomize overlays for dev and prod"
```

---

### Task 6: ArgoCD Applications (app-of-apps + platform)

**Files:**
- Create: `argocd/root-app.yaml`
- Create: `argocd/apps/app-dev.yaml`
- Create: `argocd/apps/app-prod.yaml`
- Create: `argocd/apps/rollouts.yaml`
- Create: `argocd/apps/ingress-nginx.yaml`
- Create: `argocd/apps/monitoring.yaml`
- Create: `argocd/apps/metrics-server.yaml`

**Step 1: Add ArgoCD Application manifests**

`argocd/root-app.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: lab-root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ORG/REPO.git
    targetRevision: main
    path: argocd/apps
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`argocd/apps/app-dev.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: lab-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ORG/REPO.git
    targetRevision: main
    path: k8s/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: lab-devops
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`argocd/apps/app-prod.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: lab-app-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ORG/REPO.git
    targetRevision: main
    path: k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: lab-devops
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`argocd/apps/rollouts.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-rollouts
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argo-rollouts
    targetRevision: 2.38.1
    helm:
      values: |
        dashboard:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argo-rollouts
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`argocd/apps/ingress-nginx.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: 4.11.1
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`argocd/apps/monitoring.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 61.3.1
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`argocd/apps/metrics-server.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metrics-server
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://kubernetes-sigs.github.io/metrics-server
    chart: metrics-server
    targetRevision: 3.12.1
  destination:
    server: https://kubernetes.default.svc
    namespace: metrics-server
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Step 2: Validate YAML**
Run: `kubectl apply --dry-run=client -f argocd/root-app.yaml`
Expected: successful dry-run (requires ArgoCD CRDs installed)

**Step 3: Commit**
Run:
```
git add argocd
git commit -m "feat: add argocd applications for platform and app"
```

---

### Task 7: GitHub Actions CI/CD

**Files:**
- Create: `.github/workflows/ci-cd.yaml`

**Step 1: Add workflow**

`.github/workflows/ci-cd.yaml`
```yaml
name: ci-cd

on:
  push:
    branches: [main]
    paths:
      - "app/**"
      - "k8s/**"
      - ".github/workflows/ci-cd.yaml"
  workflow_dispatch: {}

jobs:
  build_push_update:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
    - uses: actions/checkout@v4

    - name: Read version
      run: echo "VERSION=$(cat app/VERSION)" >> $GITHUB_ENV

    - name: Docker login
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Build and push
      uses: docker/build-push-action@v6
      with:
        context: ./app
        push: true
        tags: |
          docker.io/${{ secrets.DOCKERHUB_USERNAME }}/lab-api:v${{ env.VERSION }}
          docker.io/${{ secrets.DOCKERHUB_USERNAME }}/lab-api:dev

    - name: Update Kustomize image tag (dev)
      run: |
        cd k8s/overlays/dev
        kustomize edit set image docker.io/DOCKERHUB_USER/lab-api:v${VERSION}
        cd ../../..
        git add k8s/overlays/dev/kustomization.yaml
        git commit -m "chore: bump dev image to v${VERSION}"
        git push
```

**Step 2: Validate workflow locally (optional)**
Run: `yamllint .github/workflows/ci-cd.yaml`
Expected: no errors

**Step 3: Commit**
Run:
```
git add .github/workflows/ci-cd.yaml
git commit -m "ci: build push and update dev overlay"
```

---

### Task 8: README finalization + demo steps

**Files:**
- Modify: `README.md`
- Modify: `docs/diagrams/architecture.mmd`

**Step 1: Expand README**

`README.md` (key sections to add)
```
## Architecture
- Kind + ArgoCD + Rollouts + ingress-nginx + Prometheus/Grafana
- Blue/Green via Service selector
- Canary via Argo Rollouts steps

## Prereqs
- kind, kubectl, helm, argocd CLI

## Setup (local Kind)
- create cluster
- install ArgoCD
- apply argocd/root-app.yaml
- wait for apps to sync

## Demo Scenarios
1) Initial sync via ArgoCD
2) Blue -> Green switch (swap patch-service-*.yaml)
3) Canary rollout (bump image tag)
4) HPA auto-scaling (load test)

## Screenshots
- ArgoCD apps
- Argo Rollouts
- Grafana dashboard
```

**Step 2: Update diagram**
Adjust `docs/diagrams/architecture.mmd` if needed to reflect app-of-apps and monitoring.

**Step 3: Commit**
Run:
```
git add README.md docs/diagrams/architecture.mmd
git commit -m "docs: add architecture and demo guide"
```

---

## Execution Handoff

Plan prepared for `docs/plans/2026-02-05-devops-gitops-lab.md`.
