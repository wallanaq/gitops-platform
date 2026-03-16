# gitops-platform

A local GitOps lab built on **Kubernetes**, exploring the full application lifecycle: branching strategy (Gitflow), CI/CD, container builds, image registry, and GitOps-based deployment via **Helm + ArgoCD + Istio**.

## Goals

- Practice Gitflow branching strategy end-to-end
- Build and publish container images to GitHub Container Registry (ghcr.io)
- Deploy applications to a local Kubernetes cluster using Helm charts
- Manage all cluster state through ArgoCD (GitOps)
- Explore Istio service mesh: sidecar injection, traffic management, observability

## Stack

| Layer             | Tool                                |
| ----------------- | ----------------------------------- |
| Local Kubernetes  | OrbStack                            |
| GitOps controller | ArgoCD                              |
| Service mesh      | Istio                               |
| Package manager   | Helm                                |
| Image registry    | GitHub Container Registry (ghcr.io) |
| CI/CD             | GitHub Actions                      |

## Repository structure

```
.
├── infra/                        # All cluster infrastructure
│   ├── argocd/
│   │   ├── values.yaml           # ArgoCD Helm values
│   │   └── gateway.yaml          # Istio Gateway + VirtualService for ArgoCD UI
│   └── istio/
│       ├── base/values.yaml      # Istio CRDs
│       ├── istiod/values.yaml    # Istio control plane
│       └── gateway/values.yaml   # Istio ingress gateway
└── apps/                         # Application source code (coming soon)
```

## Prerequisites

- [OrbStack](https://orbstack.dev) with Kubernetes enabled
- `helm` >= 4.x
- `kubectl`
- `git`

## Bootstrap

ArgoCD is the only component installed manually. Everything else is managed by ArgoCD after bootstrap.

### 1. Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 9.4.11 \
  --values infra/argocd/values.yaml \
  --wait
```

### 2. Install Istio

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update

helm upgrade --install istio-base istio/base \
  --namespace istio-system --create-namespace \
  --version 1.29.1 --values infra/istio/base/values.yaml --wait

helm upgrade --install istiod istio/istiod \
  --namespace istio-system \
  --version 1.29.1 --values infra/istio/istiod/values.yaml --wait

helm upgrade --install istio-ingressgateway istio/gateway \
  --namespace istio-system \
  --version 1.29.1 --values infra/istio/gateway/values.yaml --wait
```

### 3. Apply ArgoCD routing

```bash
kubectl apply -f infra/argocd/gateway.yaml
```

### 4. Get the initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

ArgoCD UI: `http://argocd.192.168.139.2.nip.io`

> After first login, change the password and delete the initial secret:
>
> ```bash
> kubectl -n argocd delete secret argocd-initial-admin-secret
> ```

## Enabling Istio sidecar injection

Label any namespace to enable automatic sidecar injection:

```bash
kubectl label namespace <namespace> istio-injection=enabled
```

Every pod created in that namespace will automatically receive the Envoy sidecar proxy.
