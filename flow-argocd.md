# Argo CD UI — Step‑by‑step Flow

This document describes a complete, UI‑focused flow to deploy the Helm chart in this repo using Argo CD and how to manage image tags, ingress and TLS from the Argo CD UI.

Prerequisites
- `kubectl` configured for the target cluster.
- Argo CD installed in the `argocd` namespace and reachable (port‑forward or LoadBalancer/Ingress).
- Your Git repository reachable by Argo CD (SSH or HTTPS). This repo contains the chart at `k8s/helm/react-portfolio` and an Application manifest at `k8s/argocd/react-portfolio-app.yaml`.
- Docker image already pushed: `bomeravi/react-portfolio:v1.2.3` (current `values.yaml`).
- DNS A record for `saroj.digi.saroj.name.np` points to your cluster/ingress IP.

1) Open Argo CD UI

- Port‑forward (local access):

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
# Open https://localhost:8080 in your browser
```

- Or expose Argo CD with an Ingress/LoadBalancer and open the provided URL.

2) Get admin password and log in

- Get initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

- Login to UI: username `admin`, password from above (or use SSO if configured).

3) Add/connect your Git repository (UI)

1. In the UI click **Settings (gear) → Repositories → Connect Repo**.
2. Choose connection type: `SSH` (recommended if `repoURL` is `git@...`) or `HTTPS`.
3. Provide `Repository URL` (e.g., `git@github.com:bomeravi/k8s-react-portfolio.git`) and credentials (SSH key or username/token). Save.

Notes:
- If your `k8s/argocd/react-portfolio-app.yaml` uses an SSH `repoURL`, prefer adding the repo via SSH in the UI.

4) Create an Application from the UI (step‑by‑step)

1. Click **+ NEW APP**.
2. Fill the form:
   - **Application Name:** react-portfolio
   - **Project:** default
   - **Repository URL:** select the repo you added
   - **Revision:** `main` (or `HEAD` / branch/tag you want)
   - **Path:** `k8s/helm/react-portfolio`
   - **Cluster:** `https://kubernetes.default.svc`
   - **Namespace:** `react-portfolio`
3. Source options (Helm):
   - You can leave the chart's `values.yaml` as the source of truth (it currently sets `image.tag: v1.2.3`).
   - To override values from the UI (no git commit), click **Edit** under **Helm** → set inline values (YAML) as shown below.
4. Sync Policy:
   - Choose **Automated** to auto‑deploy changes; enable **Prune** and **Self Heal** if desired.
5. Click **Create**.

5) Override `image.tag` from the UI (no Git commit)

- Example inline Helm Values to set in the Application source (Helm → Values):

```yaml
image:
  repository: bomeravi/react-portfolio
  tag: v1.2.3
imagePullSecrets:
  - name: regcred   # if using a private image
```

- After saving values, click **SYNC** on the Application page to apply the change.

6) Enable Ingress and TLS from the UI (we already updated the chart values)

- The chart's `values.yaml` now enables ingress for `saroj.digi.saroj.name.np` and requests TLS via cert‑manager (ClusterIssuer `letsencrypt-prod`). To manage or change the host or TLS secret from UI, edit the Helm values in the Application and update the `ingress.hosts` and `ingress.tls` blocks.

Example values snippet to ensure ingress + TLS:

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: saroj.digi.saroj.name.np
      paths:
        - path: /
          pathType: Prefix
  tls:
    - hosts:
        - saroj.digi.saroj.name.np
      secretName: saroj-tls
```

Save then **SYNC**.

7) Add image pull secret (if Docker Hub private)

- Create secret in target namespace `react-portfolio`:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<DH_USER> --docker-password=<DH_PASS> --docker-email=<EMAIL> \
  -n react-portfolio
```

- Add to Helm values (UI edit) under `imagePullSecrets` as shown above, then **SYNC**.

8) Sync and monitor deployment

- In the Application page click **SYNC** → choose options → **Synchronize**.
- Watch **Resources** list and **Events** on each resource. Click a Pod to view **Logs** and **Events**.

9) Verify Ingress and Certificate

- Check ingress and cert status from the cluster (CLI):

```bash
kubectl -n react-portfolio get ingress
kubectl -n react-portfolio describe ingress <ingress-name>
kubectl -n cert-manager get certificates,orders -o wide
kubectl -n react-portfolio get secret saroj-tls -o yaml
```

- In the UI, click the Ingress resource to inspect annotations and events.

10) Troubleshooting common errors (UI + CLI)

- ImagePullBackOff:
  - UI: click Pod → Logs & Events to see image pull errors.
  - Fix: ensure `image.repository` and `image.tag` are correct and `imagePullSecrets` present for private registries.

- Certificate not issued:
  - UI: check `Certificate` resource under `react-portfolio` or `cert-manager` namespace and inspect events.
  - CLI: `kubectl -n cert-manager logs deploy/cert-manager` and `kubectl describe challenge <name>`.

- App OutOfSync after Git change:
  - If you change `values.yaml` in the repo, Argo CD will show the Application as OutOfSync; click **SYNC** to apply.
  - If you want Argo CD to auto-apply commits, ensure **Automated** sync is enabled.

11) Rollback and history

- Use the UI: Application → **History and Rollback** to view previous revisions and rollback to a prior Git commit/revision.

12) CLI equivalents (for reference)

```bash
# Add repo (example via argocd CLI)
argocd repo add git@github.com:bomeravi/k8s-react-portfolio.git --ssh-private-key-path ~/.ssh/id_rsa

# Create app via CLI (with helm override)
argocd app create react-portfolio \
  --repo git@github.com:bomeravi/k8s-react-portfolio.git --path k8s/helm/react-portfolio --dest-server https://kubernetes.default.svc --dest-namespace react-portfolio \
  --helm-set image.tag=v1.2.3

# Sync
argocd app sync react-portfolio

# Set an override to change tag without committing
argocd app set react-portfolio --helm-set image.tag=v1.2.3 && argocd app sync react-portfolio
```

13) Files in this repo to review from the UI
- `k8s/argocd/react-portfolio-app.yaml` — existing Application manifest (you can apply it instead of creating an app in UI).
- `k8s/helm/react-portfolio/values.yaml` — image repo/tag, ingress, and TLS settings (already set to `v1.2.3` and `saroj.digi.saroj.name.np`).
- `k8s/helm/react-portfolio/templates/ingress.yaml` — ingress template used by the chart.

Best practices and notes
- Prefer Git as the single source of truth: commit `values.yaml` changes when possible so the repo reflects the desired state.
- Use Argo CD inline Helm overrides only for temporary or emergency changes; inline overrides are stored in the Argo CD Application resource, not in Git.
- Use `cert-manager` with a ClusterIssuer for automated Let's Encrypt certificates; ensure your ingress controller handles `http01` challenges.

Next steps (optional)
- I can apply the Application manifest (`kubectl apply -f k8s/argocd/react-portfolio-app.yaml -n argocd`) and perform an initial UI sync for you.
- I can add the `ClusterIssuer` manifest file to `k8s/cert-manager/clusterissuer-letsencrypt.yaml` and commit it.
