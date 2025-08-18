## Reddit Clone App

A simple Reddit-like web application built with Next.js, React, Chakra UI, Recoil, and Firebase. This app is containerized with Docker and integrated into a CI/CD pipeline with Jenkins, SonarQube, and Trivy. Kubernetes deployment manifests and the CD pipeline live in the sibling GitOps directory.

- **GitOps assets**: see `reddit-gitOps` → [k8s manifests and CD](../reddit-gitOps)

### Tech stack
- **Frontend**: Next.js 12, React 17, Chakra UI, Recoil
- **Backend services**: Firebase (Auth, Firestore, Storage) and Firebase Cloud Functions
- **CI/CD**: Jenkins pipelines, SonarQube analysis, Trivy scans, Docker Hub image push
- **Runtime**: Docker, Kubernetes

### Repo layout
- `redit-clone-app` (this folder): Next.js app, Dockerfile, CI pipeline (`Jenkinsfile`)
- `reddit-gitOps`: Kubernetes manifests and CD pipeline for deploying the built image
  - `k8s/deployment.yml`
  - `k8s/service.yml`
  - `Jenkinsfiles` (CD job in Jenkins)

## Getting started (local development)

Prerequisites:
- Node.js 16+ and npm

Install and run:

```bash
npm install
npm run dev
```

Open the app at `http://localhost:3000`.

### Firebase configuration
The sample Firebase project configuration is currently hardcoded in `src/firebase/clientApp.ts` under `firebaseConfig`. To use your own Firebase project, replace the values in that file with your project's web app config from Firebase Console.

## Scripts
- `npm run dev`: start Next.js in development
- `npm run build`: build for production
- `npm run start`: run the production build
- `npm run lint`: run ESLint

## Docker

Build the image:
```bash
docker build -t <your-dockerhub-user>/reddit-clone-app:<tag> .
```

Run the container:
```bash
docker run --rm -p 3000:3000 <your-dockerhub-user>/reddit-clone-app:<tag>
```

Note: The provided `Dockerfile` currently runs `npm run dev`. For production containers, consider updating it to build the app and run `next start` instead.

## CI pipeline (Jenkins)

The CI pipeline resides in `redit-clone-app/Jenkinsfile` and performs:
- Workspace clean and optional repo checkout
- SonarQube analysis
- `npm install` and `npm audit fix`
- Trivy filesystem scan
- Docker login, build, and push to Docker Hub
- Trivy image scan
- Cleanup of local image
- Trigger of the CD pipeline with the built `IMAGE_TAG`

Key environment variables and credentials referenced by the pipeline:
- `SONAR_TOKEN` (credential id: `sonar-token`)
- Docker Hub credentials (credential id: `dockerhub`)
- `JENKINS_API_TOKEN` for triggering the CD job

Image naming in CI:
- `IMAGE_NAME = <docker-user>/reddit-clone-app`
- `IMAGE_TAG = <release>-<BUILD_NUMBER>`

## Deployment (Kubernetes via GitOps)

Kubernetes manifests and the CD pipeline are in `reddit-gitOps`:
- Manifests: [../reddit-gitOps/k8s](../reddit-gitOps/k8s)
- Deployment: updates the container image to the tag built by CI (`spec.template.spec.containers[0].image`)

How deployment flows end-to-end:
- CI builds and pushes an image, then the CD job in `reddit-gitOps/Jenkinsfiles` updates `k8s/deployment.yml` with the new `IMAGE_TAG` and pushes to the GitOps repo.
- Argo CD continuously watches the GitOps repo and automatically syncs changes to the cluster (no manual `kubectl apply` or `kubectl rollout` is required).

Example files to review/update:
- `deployment.yml` → image: `rahul6364/reddit-clone-app:1.0.0-<build>`
- `service.yml` → exposes port `3000` as `ClusterIP`

Optional: manual kubectl fallback (not needed with Argo CD):
```bash
kubectl apply -f ../reddit-gitOps/k8s/deployment.yml
kubectl apply -f ../reddit-gitOps/k8s/service.yml
```

Check status (Argo CD):
```bash
# If using Argo CD CLI (replace server if exposed differently)
argocd login localhost:8080 --username admin --password <token-or-password> --insecure
argocd app get reddit-clone
argocd app history reddit-clone
# Manual sync only if you disabled auto-sync
argocd app sync reddit-clone
```

Optional: verify with kubectl:
```bash
kubectl rollout status deployment/reddit-clone-deployment
kubectl get svc reddit-clone-service
```

Since the Service is `ClusterIP`, access internally or port-forward for local testing:
```bash
kubectl port-forward svc/reddit-clone-service 3000:3000
# then open http://localhost:3000
```

If you need external access, create an Ingress or change Service type to `NodePort`/`LoadBalancer` per your cluster setup.

### Argo CD (GitOps automation)
If you use Argo CD to reconcile cluster state from the GitOps repo, point an `Application` at the `reddit-gitOps` repository path.

Quick start (install Argo CD):
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Example `Application` (replace repo URL/paths/namespaces as needed):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: reddit-clone
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/rahul6364/reddit-gitOps.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Access Argo CD UI (port-forward):
```bash
kubectl -n argocd port-forward svc/argocd-server 8080:80
# open http://localhost:8080
```

## Firebase Cloud Functions (optional)
Cloud functions are in `functions/` and include user document creation and comment cleanup on post delete. Deploy with Firebase CLI if you use them in your environment.

## Monitoring (Prometheus & Grafana)
If you installed Prometheus and Grafana (e.g., via `kube-prometheus-stack` Helm chart) to monitor the app/cluster, typical access patterns are:

- Port-forward Prometheus:
```bash
kubectl -n monitoring port-forward svc/prometheus-k8s 9090:9090 || \
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```

- Port-forward Grafana:
```bash
kubectl -n monitoring port-forward svc/grafana 3001:80 || \
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3001:80
```

Log in to Grafana at `http://localhost:3001` and import dashboards or use built-in Kubernetes dashboards. Adjust Service names to match your installation.

## Links
- App CI pipeline: `redit-clone-app/Jenkinsfile`
- GitOps (CD + Kubernetes): [../reddit-gitOps](../reddit-gitOps)

