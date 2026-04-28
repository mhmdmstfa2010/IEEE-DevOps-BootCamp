# IEEE DevOps Bootcamp - Voting App Demo

Beginner-friendly, step-by-step training repository for:

- Docker
- Docker Compose
- Docker Hub
- Kubernetes (kind)
- GitHub Actions (CI)
- Argo CD (GitOps CD)

Based on: [dockersamples/example-voting-app](https://github.com/dockersamples/example-voting-app)

## Architecture

![Architecture diagram](architecture.excalidraw.png)

- `vote` (Python): voting UI
- `redis`: temporary queue
- `worker` (.NET): reads votes and writes to DB
- `db` (PostgreSQL): persistent storage
- `result` (Node.js): live results UI

## 1) Windows setup

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

Starter workflow included at:

- `.github/workflows/bootcamp-ci.yml`

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

Open `https://localhost:8085`

Get initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Use:

- Username: `admin`
- Password: command output

Apply the included Argo CD app template:

```bash
kubectl apply -f argocd-app.yaml
```

Before applying, update `argocd-app.yaml` with your GitHub repo URL.

## 10) End of workshop cleanup

```bash
kubectl delete namespace vote
kind delete cluster --name voting-cluster
docker compose down -v
```

## 11) Day2 Kubernetes Topic-to-Demo Mapping

Use this section while teaching: each Kubernetes topic is followed by a direct live practice on the voting app.

### 11.1 Core objects: Node, Namespace, Pod, Deployment, ReplicaSet, Service

```bash
kubectl get nodes
kubectl get ns
kubectl get all -n vote
kubectl get deploy,rs,pods,svc -n vote
```

### 11.2 Imperative vs Declarative

Imperative:

```bash
kubectl create deployment demo-nginx --image=nginx
kubectl get deploy demo-nginx
kubectl delete deployment demo-nginx
```

Declarative:

```bash
kubectl apply -f k8s-specifications/ -n vote
```

### 11.3 Inspect pod lifecycle and troubleshooting

```bash
kubectl get pods -n vote
kubectl describe pod -n vote <pod-name>
kubectl logs -n vote <pod-name>
kubectl exec -it -n vote <pod-name> -- sh
```

### 11.4 Self-healing and scaling

```bash
kubectl scale deployment vote --replicas=3 -n vote
kubectl get pods -n vote -o wide
kubectl delete pod -n vote <one-vote-pod>
kubectl get pods -n vote
```

### 11.5 Rolling update and rollback

```bash
kubectl set image deployment/vote vote=nginx:latest -n vote
kubectl rollout status deployment/vote -n vote
kubectl rollout history deployment/vote -n vote
kubectl rollout undo deployment/vote -n vote
```

### 11.6 Service types and networking

```bash
kubectl get svc -n vote
kubectl describe svc vote -n vote
kubectl describe svc result -n vote
```

### 11.7 Ingress on the voting app

Install ingress controller first, then apply this ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: voting-ingress
  namespace: vote
spec:
  ingressClassName: nginx
  rules:
    - host: voting.local
      http:
        paths:
          - path: /vote
            pathType: Prefix
            backend:
              service:
                name: vote
                port:
                  number: 5000
          - path: /result
            pathType: Prefix
            backend:
              service:
                name: result
                port:
                  number: 5001
```

### 11.8 Multi-container pod (shared volume demo)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers-demo
  namespace: vote
spec:
  restartPolicy: Never
  volumes:
    - name: shared-area
      emptyDir: {}
  containers:
    - name: nginx-container
      image: nginx
      volumeMounts:
        - name: shared-area
          mountPath: /usr/share/nginx/html
    - name: helper-container
      image: busybox
      command: ["/bin/sh", "-c"]
      args: ["echo Hello from helper container > /shared/index.html && sleep 3600"]
      volumeMounts:
        - name: shared-area
          mountPath: /shared
```

### 11.9 PV and PVC

```bash
kubectl get pv,pvc -n vote
kubectl describe deployment db -n vote
kubectl get storageclass
```

### 11.10 ConfigMap

```bash
kubectl create configmap vote-config -n vote \
  --from-literal=APP_TITLE="IEEE Bootcamp Voting App" \
  --from-literal=APP_THEME="blue"
kubectl get configmap vote-config -n vote -o yaml
```

### 11.11 Secret

```bash
kubectl create secret generic db-secret -n vote \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=postgres
kubectl get secret db-secret -n vote
kubectl describe secret db-secret -n vote
```

### 11.12 Stateful vs Stateless and StatefulSet

- Stateless in this app: `vote`, `result`, `worker`
- Stateful in this app: `db`

```bash
kubectl get statefulset -A
kubectl get pvc -A
```

### 11.13 Jobs and CronJobs

```bash
kubectl create job vote-once --image=busybox -- /bin/sh -c "echo one-time task; date"
kubectl get jobs
kubectl logs job/vote-once
kubectl create cronjob vote-cron --image=busybox --schedule="*/5 * * * *" -- /bin/sh -c "date; echo bootcamp"
kubectl get cronjobs
```
