# DevOps GitOps Lab

## Architecture
- Kind local cluster (Kubernetes).
- ArgoCD app-of-apps deploys platform components and app overlays.
- Blue/Green via dual Deployments and a Service selector flip.
- Canary via Argo Rollouts with 10/30/50/100 weights and pauses.
- Prometheus + Grafana for metrics; HPA scales on CPU.

## Repo Structure
- `app/` FastAPI app with /healthz, /version, /metrics.
- `k8s/base/` base manifests (Deployments, Rollout, Services, HPA, Ingress).
- `k8s/overlays/dev/` and `k8s/overlays/prod/` Kustomize overlays.
- `argocd/` ArgoCD Application CRs (root + apps).
- `.github/workflows/` CI/CD pipeline.

## Prereqs
- kind, kubectl, helm, argocd, docker
- optional: kustomize, hey or k6

## Setup (local Kind)
1) Create cluster
```bash
kind create cluster --name devops-lab
```

2) Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

3) Update placeholders
- Replace `ORG/REPO` in `argocd/root-app.yaml` with your GitHub repo.
- Atualize o repositório da imagem caso use outro Docker Hub user (atual: `brenofrancoa`).
- Add `lab.local` to `/etc/hosts` pointing to the ingress IP (or use port-forward).

4) Apply root app
```bash
kubectl apply -f argocd/root-app.yaml
```

5) Access ArgoCD
```bash
kubectl -n argocd port-forward svc/argocd-server 8080:80
```

## GitOps Flow
1) Commit updates in `app/`.
2) GitHub Actions builds and pushes image to Docker Hub.
3) Workflow updates `k8s/overlays/dev/kustomization.yaml` with the new tag.
4) ArgoCD auto-sync deploys the new version.

## Demo Scenarios
1) Initial sync via ArgoCD.
2) Blue -> Green: switch the Service selector patch in overlays.
3) Canary rollout: bump image tag and observe steps.
4) HPA: run a load test and watch replicas scale.

Example load test:
```bash
hey -z 2m -c 50 http://lab.local/canary/healthz
```

## CI/CD
- Workflow: `.github/workflows/ci-cd.yaml`
- Secrets: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`

## Screenshots
Add images to `docs/screenshots/`:
- `argocd-apps.png`
- `argocd-rollouts.png`
- `grafana-dashboard.png`
