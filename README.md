# React Portfolio — Argo CD + Helm Deployment

This repository contains a Helm chart and an Argo CD Application manifest to deploy the React portfolio image from Docker Hub.

Quick overview
- Build and push your Docker image to Docker Hub: `<DH_USER>/react-portfolio:<TAG>`
- Update or let CI update `k8s/helm/react-portfolio/values.yaml` image tag.
- Argo CD reads the chart at `k8s/helm/react-portfolio` and will deploy the image tag from `values.yaml`.

Commands

# If you need to install locally with Helm (optional):
helm install react-portfolio k8s/helm/react-portfolio -n react-portfolio --create-namespace

# Apply Argo CD Application (Argo CD will manage Helm releases):
kubectl apply -f k8s/argocd/react-portfolio-app.yaml -n argocd

# Sync (optionally via argocd CLI):
argocd app sync react-portfolio

How to set the image tag
- Edit `k8s/helm/react-portfolio/values.yaml` and set:

  image:
    repository: <DH_USER>/react-portfolio
    tag: "<TAG>"

- Commit and push; Argo CD will detect the change and deploy the new image (or use `argocd app sync`).

Private images
- Create a Kubernetes docker-registry secret in the target namespace and add it as `imagePullSecrets` in the Helm chart or values.yaml:

  kubectl create secret docker-registry regcred \
    --docker-server=https://index.docker.io/v1/ \
    --docker-username=<DH_USER> --docker-password=<PASSWORD> --docker-email=<EMAIL> -n react-portfolio

CI integration (Jenkins)
- The included `Jenkinsfile` updates `k8s/helm/react-portfolio/values.yaml` with the new tag and pushes the change back to the repo.
- Ensure the Jenkins job has push credentials and that branch names match (the pipeline currently pushes to `main`).

Verification
- Check Argo CD UI or run `kubectl -n react-portfolio get deploy,po -l app.kubernetes.io/name=react-portfolio` and `kubectl -n react-portfolio describe pod <pod>` / `kubectl -n react-portfolio logs <pod>`.

Notes and recommendations
- The Helm chart templates already reference `.Values.image.repository` and `.Values.image.tag`.
- If you prefer Argo CD to override the tag instead of editing `values.yaml`, update the Application manifest with `spec.source.helm.values`.
- If using SSH for `repoURL` in the Application, ensure Argo CD has the SSH key configured.

Ingress & TLS (Let's Encrypt)
- Requirements: an Ingress controller (e.g. nginx-ingress) and `cert-manager` installed in the cluster. Your domain DNS must point to the Ingress controller's external IP (you mentioned the A record is set to the Kubernetes master IP — ensure that routes to the ingress controller).

1) Install nginx ingress (example using the community chart):

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

2) Install cert-manager:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace
```

3) Create a ClusterIssuer for Let's Encrypt (production) — replace `<EMAIL>` with your email:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <EMAIL>
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
```

Save that as `k8s/cert-manager/clusterissuer-letsencrypt.yaml` and apply:

```bash
kubectl apply -f k8s/cert-manager/clusterissuer-letsencrypt.yaml
```

4) The Helm chart `values.yaml` has been updated to enable ingress and request TLS via `cert-manager` using the `letsencrypt-prod` ClusterIssuer. Argo CD will pick up the changes; run a sync in the UI or CLI:

```bash
argocd app sync react-portfolio
```

5) Verify certificate issuance and Ingress:

```bash
kubectl -n react-portfolio get ingress
kubectl -n react-portfolio describe certificate
kubectl -n cert-manager get orders,certificates
kubectl -n react-portfolio get secret saroj-tls -o yaml
```

If certificate issuance fails, check cert-manager logs and `kubectl describe` the Challenge/Order resources for details.
