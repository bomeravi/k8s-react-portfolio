# Kubernetes Manifests

This repo keeps all Kubernetes and Argo CD files under `k8s/`.

Directories:
- `k8s/helm/react-portfolio`: Helm chart (recommended for production).
- `k8s/argocd/react-portfolio-app.yaml`: Argo CD Application manifest pointing at the Helm chart.
- `k8s/kustomize`: Kustomize base + prod overlay (example alternative).
- `k8s/manifests`: Plain YAML manifests (example alternative).

**End-to-End Flow**
1. Jenkins builds and pushes the Docker image.
2. Jenkins updates the Helm image tag in this repo and pushes the change.
3. Argo CD detects the Git change and syncs the release to the cluster.

**Prereqs**
- A Kubernetes cluster and Argo CD installed.
- Jenkins with Docker access.
- Docker registry credentials stored in Jenkins as `dockerhub-creds` (username/password).
- Git SSH credentials stored in Jenkins as `k8s-git-ssh` (SSH key with push access to `git@github.com:bomeravi/k8s-react-portfolio.git`).
- Argo CD repo credentials to read `git@github.com:bomeravi/k8s-react-portfolio.git` if the repo is private.

**Jenkins Pipeline (App Repo)**
1. Create a Jenkins pipeline for the app repository using the root `Jenkinsfile`.
2. Trigger a build with `IMAGE_TAG` or leave it empty to use `v${BUILD_NUMBER}`.
3. The pipeline builds and pushes `IMAGE_REPO:IMAGE_TAG`, then updates `k8s/helm/react-portfolio/values.yaml` (`image.tag`) and `k8s/helm/react-portfolio/Chart.yaml` (`appVersion`).
4. The pipeline commits and pushes those changes to this repo, which triggers Argo CD.

The root `Jenkinsfile` uses these defaults, which you can change if needed:
- `K8S_REPO_URL`: `git@github.com:bomeravi/k8s-react-portfolio.git`
- `K8S_REPO_BRANCH`: `main`
- `K8S_GIT_CREDENTIALS_ID`: `k8s-git-ssh`
- `K8S_VALUES_FILE`: `k8s/helm/react-portfolio/values.yaml`
- `K8S_CHART_FILE`: `k8s/helm/react-portfolio/Chart.yaml`

**Argo CD Setup**
1. Add the repo to Argo CD using an SSH key or repo credential.
2. Apply the Argo CD Application manifest:

```bash
kubectl apply -f k8s/argocd/react-portfolio-app.yaml
```

3. Argo CD will auto-sync because `syncPolicy.automated` is enabled in the Application.

**Helm (Manual) Option**
If you are not using Argo CD, you can deploy directly with Helm:

```bash
helm upgrade --install react-portfolio k8s/helm/react-portfolio \
  --namespace react-portfolio \
  --create-namespace \
  --set image.repository=bomeravi/react-portfolio \
  --set image.tag=v1.2.3
```

**Optional Jenkinsfile (K8s Repo)**
There is also a Jenkins pipeline in `k8s-react-portfolio/Jenkinsfile` if you want a standalone job that only updates the Helm image tag in this repo.
