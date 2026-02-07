# Kubernetes Manifests

This repo keeps all Kubernetes and Argo CD files under `k8s/`.

Directories:
- `k8s/helm/react-portfolio`: Helm chart (recommended for production).
- `k8s/argocd/react-portfolio-app.yaml`: Argo CD Application manifest pointing at the Helm chart.
- `k8s/kustomize`: Kustomize base + prod overlay (example alternative).
- `k8s/manifests`: Plain YAML manifests (example alternative).

Recommended for production:
- Helm or Kustomize with GitOps (Argo CD). Helm is usually preferred when you need parameterized releases and versioned templates.

To deploy with Argo CD, apply the Application manifest:

```bash
kubectl apply -f k8s/argocd/react-portfolio-app.yaml
```
