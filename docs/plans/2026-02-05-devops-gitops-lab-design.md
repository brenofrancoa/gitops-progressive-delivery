# DevOps GitOps Lab Design

## Summary
A production-like GitOps lab on a local Kind cluster using ArgoCD and Argo Rollouts. The lab demonstrates Blue/Green and Canary deployments, Prometheus/Grafana monitoring, and HPA autoscaling. The repository is a mono-repo with FastAPI app code and Kubernetes manifests organized via Kustomize.

## Goals
- Provide a portfolio-grade GitOps lab with clear documentation and demos.
- Demonstrate Blue/Green traffic switching with zero downtime.
- Demonstrate Canary rollout with staged traffic weights and pauses.
- Provide observability with Prometheus and Grafana.
- Demonstrate autoscaling via HPA under load.

## Non-goals
- Production-grade security hardening (RBAC fine-tuning, secrets management).
- Multi-cluster federation or advanced service mesh features.
- Full Terraform provisioning (optional future enhancement).

## Architecture
- Kubernetes: Kind (local).
- GitOps: ArgoCD auto-sync with prune and self-heal.
- Progressive delivery: Argo Rollouts for Canary; dual Deployments + Service selector for Blue/Green.
- Observability: kube-prometheus-stack (Prometheus + Grafana).
- Autoscaling: metrics-server + HPA based on CPU utilization.
- Ingress: ingress-nginx with path-based routing.

## Repository Layout
- /app: FastAPI application with /healthz, /version, /metrics.
- /k8s/base: Base manifests and shared configuration.
- /k8s/overlays/dev: Dev overlay with image tag and service selector.
- /k8s/overlays/prod: Prod overlay with image tag and service selector.
- /argocd: ArgoCD Application CRs and app-of-apps root.
- /.github/workflows: CI/CD pipeline.
- /docs: diagrams and screenshots placeholders.

## Deployment Strategies
### Blue/Green
- Two Deployments: api-blue and api-green.
- Single Service: api-bg.
- Traffic switch by updating Service selector from color=blue to color=green.

### Canary
- Rollout object with steps: 10%, 30%, 50%, 100%.
- Pauses between steps to observe behavior and metrics.
- Stable and canary Services created for Rollouts.

## GitOps Flow
- Repo is the source of truth.
- ArgoCD Applications deploy platform components and app overlays.
- Image tag updates in overlays trigger auto-sync deployments.

## CI/CD
- GitHub Actions builds and pushes Docker images to Docker Hub.
- Semantic version stored in /app/VERSION.
- Workflow updates kustomization image tag in overlays and commits changes.

## Demo Scenarios
1) Initial ArgoCD sync shows all apps healthy.
2) Blue -> Green switch by patching Service selector.
3) Canary rollout by bumping image tag and observing steps.
4) HPA scales up during load test with hey or k6.

## Open Decisions
- Optional: add Prometheus Adapter for custom metrics.
- Optional: add Terraform for Kind or cloud cluster provisioning.
