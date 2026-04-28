# IEEE DevOps Bootcamp - Voting App Demo

Beginner-friendly training repository for:

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

## 0) One-time Setup (Windows)

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

Clone:

```bash
git clone https://github.com/dockersamples/example-voting-app.git
cd example-voting-app
```

---

## Day 1 - Docker Fundamentals

### 1) Docker only (manual containers)

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

### 2) Docker Compose

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

### 3) Push images to Docker Hub

```bash
docker login
docker push YOUR_DOCKERHUB_USERNAME/vote:v1
docker push YOUR_DOCKERHUB_USERNAME/result:v1
docker push YOUR_DOCKERHUB_USERNAME/worker:v1
```

---

## Day 2 - Kubernetes Deep Dive (All practice on voting app)

### 1) Create kind cluster

```bash
kind create cluster --config kind-config.yaml
kubectl cluster-info
kubectl get nodes
```

### 2) Deploy voting app baseline

```bash
kubectl create namespace vote
kubectl apply -f k8s-specifications/ -n vote
kubectl get pods -n vote -w
```

```bash
kubectl set image deployment/vote vote=YOUR_DOCKERHUB_USERNAME/vote:v1 -n vote
kubectl set image deployment/result result=YOUR_DOCKERHUB_USERNAME/result:v1 -n vote
kubectl set image deployment/worker worker=YOUR_DOCKERHUB_USERNAME/worker:v1 -n vote
```

Open:

- Vote app: http://localhost:31000
- Result app: http://localhost:31001

### 3) Core objects: Node, Namespace, Pod, Deployment, ReplicaSet, Service

```bash
kubectl get nodes
kubectl get ns
kubectl get all -n vote
kubectl get deploy,rs,pods,svc -n vote
```

### 4) Imperative vs Declarative

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

### 5) Pod lifecycle and troubleshooting

```bash
kubectl get pods -n vote
kubectl describe pod -n vote <pod-name>
kubectl logs -n vote <pod-name>
kubectl exec -it -n vote <pod-name> -- sh
```

### 6) Self-healing and scaling

```bash
kubectl scale deployment vote --replicas=3 -n vote
kubectl get pods -n vote -o wide
kubectl delete pod -n vote <one-vote-pod>
kubectl get pods -n vote
```

### 7) Rolling update and rollback

```bash
kubectl set image deployment/vote vote=nginx:latest -n vote
kubectl rollout status deployment/vote -n vote
kubectl rollout history deployment/vote -n vote
kubectl rollout undo deployment/vote -n vote
```

### 8) Service types and networking

```bash
kubectl get svc -n vote
kubectl describe svc vote -n vote
kubectl describe svc result -n vote
```

### 9) Ingress on voting app

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

### 10) Multi-container pod (shared volume demo)

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

### 11) PV and PVC

```bash
kubectl get pv,pvc -n vote
kubectl describe deployment db -n vote
kubectl get storageclass
```

### 12) ConfigMap

```bash
kubectl create configmap vote-config -n vote \
  --from-literal=APP_TITLE="IEEE Bootcamp Voting App" \
  --from-literal=APP_THEME="blue"
kubectl get configmap vote-config -n vote -o yaml
```

### 13) Secret

```bash
kubectl create secret generic db-secret -n vote \
  --from-literal=POSTGRES_USER=postgres \
  --from-literal=POSTGRES_PASSWORD=postgres
kubectl get secret db-secret -n vote
kubectl describe secret db-secret -n vote
```

### 14) Stateful vs Stateless and StatefulSet

- Stateless in this app: `vote`, `result`, `worker`
- Stateful in this app: `db`

```bash
kubectl get statefulset -A
kubectl get pvc -A
```

### 15) Jobs and CronJobs

```bash
kubectl create job vote-once --image=busybox -- /bin/sh -c "echo one-time task; date"
kubectl get jobs
kubectl logs job/vote-once
kubectl create cronjob vote-cron --image=busybox --schedule="*/5 * * * *" -- /bin/sh -c "date; echo bootcamp"
kubectl get cronjobs
```

---

## Day 3 - CI/CD and GitOps

### 1) Traditional CI/CD pipeline (direct deploy)

Flow:

1. Developer pushes code
2. GitHub Actions builds and pushes images
3. Same pipeline deploys directly to cluster using `kubectl set image`

Workflow:

- `.github/workflows/traditional-cicd.yml`

Required GitHub secrets:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
- `KUBE_CONFIG_DATA` (base64 of kubeconfig file)

Generate `KUBE_CONFIG_DATA`:

```bash
base64 -w 0 ~/.kube/config
```

### 2) GitOps pipeline (Argo CD pull model)

Flow:

1. Developer pushes code
2. GitHub Actions builds and pushes images
3. Pipeline updates Kubernetes manifests in Git and pushes commit
4. Argo CD detects Git change and syncs to cluster

Workflow:

- `.github/workflows/gitops-cicd.yml`

Required GitHub secrets:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`

### 3) Install Argo CD

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

### 4) Run a fair performance comparison

Use the same app change for both pipelines (for example add one line in `vote/app.py`), then compare:

- **Metric A (Pipeline duration):**
  - From GitHub Actions run page:
    - traditional workflow total duration
    - gitops workflow total duration
- **Metric B (Time to live app update):**
  - Start timer at git push
  - Stop timer when new app version is available on `http://localhost:31000`
- **Metric C (Operational model):**
  - Traditional: CI tool needs direct cluster credentials
  - GitOps: ArgoCD in cluster pulls from Git (no direct deploy from CI)

Suggested result table:

| Model | Build+Push time | Deploy time | Total time | Cluster credential in CI |
|---|---:|---:|---:|---|
| Traditional CI/CD | X min | Y min | X+Y min | Yes |
| GitOps (ArgoCD) | X min | Z min (sync delay) | X+Z min | No |

---

## Final Cleanup

```bash
kubectl delete cronjob vote-cron -n default --ignore-not-found
kubectl delete job vote-once -n default --ignore-not-found
kubectl delete namespace vote --ignore-not-found
kubectl delete namespace argocd --ignore-not-found
kind delete cluster --name voting-cluster
docker compose down -v
```
