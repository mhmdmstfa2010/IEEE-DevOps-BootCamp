# DevOps Bootcamp Demo (Beginner Friendly)

This guide turns the Example Voting App into a full bootcamp demo:

1. Docker only
2. Docker Compose
3. Push images to Docker Hub
4. Kubernetes on local cluster (kind)
5. CI with GitHub Actions
6. CD with Argo CD (GitOps)

Repository used in class: [dockersamples/example-voting-app](https://github.com/dockersamples/example-voting-app)

## 1) Setup for Windows students

Install:

- Git for Windows (Git Bash)
- Docker Desktop
- kubectl
- kind

Quick install with PowerShell (Admin):

```powershell
winget install --id Git.Git -e
winget install --id Docker.DockerDesktop -e
winget install --id Kubernetes.kubectl -e
winget install --id Kubernetes.kind -e
```

Verify in Git Bash:

```bash
git --version
docker --version
kubectl version --client
kind version
```

## 2) Clone project

```bash
git clone https://github.com/dockersamples/example-voting-app.git
cd example-voting-app
```

## 3) Demo A: Docker only (manual containers)

Use your own Docker Hub username in all tags.

```bash
docker build -t YOUR_DOCKERHUB_USERNAME/vote:v1 ./vote
docker build -t YOUR_DOCKERHUB_USERNAME/result:v1 ./result
docker build -t YOUR_DOCKERHUB_USERNAME/worker:v1 ./worker
```

```bash
docker network create vote-net
docker volume create db-data
```

```bash
docker run -d --name redis --network vote-net redis:alpine
docker run -d --name db --network vote-net -e POSTGRES_HOST_AUTH_METHOD=trust -v db-data:/var/lib/postgresql/data postgres:15-alpine
docker run -d --name worker --network vote-net YOUR_DOCKERHUB_USERNAME/worker:v1
docker run -d --name vote --network vote-net -p 8080:80 YOUR_DOCKERHUB_USERNAME/vote:v1
docker run -d --name result --network vote-net -p 8081:80 YOUR_DOCKERHUB_USERNAME/result:v1
```

Open:

- Vote app: http://localhost:8080
- Result app: http://localhost:8081

Cleanup:

```bash
docker rm -f vote result worker redis db
docker network rm vote-net
```

## 4) Demo B: Docker Compose

```bash
docker compose up --build -d
docker compose ps
```

Open:

- Vote app: http://localhost:8080
- Result app: http://localhost:8081

```bash
docker compose down
```

## 5) Demo C: Push images to Docker Hub

```bash
docker login
docker push YOUR_DOCKERHUB_USERNAME/vote:v1
docker push YOUR_DOCKERHUB_USERNAME/result:v1
docker push YOUR_DOCKERHUB_USERNAME/worker:v1
```

## 6) Demo D: Create Kubernetes cluster with kind

Create `kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: voting-cluster
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 31000
        hostPort: 31000
        protocol: TCP
      - containerPort: 31001
        hostPort: 31001
        protocol: TCP
```

Create cluster:

```bash
kind create cluster --config kind-config.yaml
kubectl cluster-info
kubectl get nodes
```

## 7) Demo E: Deploy app to Kubernetes

```bash
kubectl create namespace vote
kubectl apply -f k8s-specifications/ -n vote
kubectl get pods -n vote -w
```

Update images to your Docker Hub images:

```bash
kubectl set image deployment/vote vote=YOUR_DOCKERHUB_USERNAME/vote:v1 -n vote
kubectl set image deployment/result result=YOUR_DOCKERHUB_USERNAME/result:v1 -n vote
kubectl set image deployment/worker worker=YOUR_DOCKERHUB_USERNAME/worker:v1 -n vote
```

Open:

- Vote app: http://localhost:31000
- Result app: http://localhost:31001

## 8) Demo F: CI with GitHub Actions

1. Fork this repo to your GitHub account.
2. Add repository secrets:
   - `DOCKERHUB_USERNAME`
   - `DOCKERHUB_TOKEN`
3. Push to `main`.
4. Check Actions tab and verify the workflow publishes images.

This repo includes a starter workflow at `.github/workflows/bootcamp-ci.yml`.

## 9) Demo G: CD with Argo CD (GitOps)

Install Argo CD:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
```

Access UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8085:443
```

Open https://localhost:8085

Get initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Use:

- Username: `admin`
- Password: command output

Apply the included Argo CD application template:

```bash
kubectl apply -f argocd-app.yaml
```

Before applying, update `argocd-app.yaml` with your GitHub repository URL.

## 10) End of workshop cleanup

```bash
kubectl delete namespace vote
kind delete cluster --name voting-cluster
docker compose down -v
```
