# URL Shortener — Multi-Service Application on Kubernetes

A microservice-based URL shortener deployed with Docker, Kubernetes (Minikube), Ingress NGINX, Horizontal Pod Autoscaler, Prometheus + Grafana monitoring, k6 load testing and GitHub Actions CI.

| | |
|---|---|
| **Submitted by** | Mahbuba Sultana Shammi |
| **Batch** | Ostad Batch-10 |
| **Course** | DevOps |
| **Assignment** | Deployment of a Multi-Service URL Shortener Application Using Kubernetes & CI/CD |
| **Submission Date** | July 2026 |

---

## Table of Contents

1. [Architecture](#architecture)
2. [Technology Stack](#technology-stack)
3. [Repository Layout](#repository-layout)
4. [Prerequisites](#prerequisites)
5. [Deployment — Part 1: Docker Compose (local)](#deployment--part-1-docker-compose-local)
6. [Deployment — Part 2: Kubernetes on Minikube](#deployment--part-2-kubernetes-on-minikube)
7. [Deployment — Part 3: Ingress](#deployment--part-3-ingress)
8. [Deployment — Part 4: Horizontal Pod Autoscaler](#deployment--part-4-horizontal-pod-autoscaler)
9. [Deployment — Part 5: Monitoring with Prometheus & Grafana](#deployment--part-5-monitoring-with-prometheus--grafana)
10. [Deployment — Part 6: Load Testing with k6](#deployment--part-6-load-testing-with-k6)
11. [CI/CD with GitHub Actions](#cicd-with-github-actions)
12. [Verification Checklist](#verification-checklist)
13. [Cleanup / Teardown](#cleanup--teardown)
14. [Troubleshooting](#troubleshooting)
15. [API Reference](#api-reference)
16. [Database Schema](#database-schema)
17. [Learning Outcomes](#learning-outcomes)
18. [Credits & License](#credits--license)

---

## Architecture

Three independent application services plus Redis. Each service owns its own SQLite database — no shared database between services.

```
                          Ingress NGINX (urlshortener.local)
                                       |
              /python          /go             /node
                 |              |                |
        +--------v-----+  +-----v------+  +------v-------+
        |   Python     |  |    Go      |  |   Node.js    |
        |  Dashboard   |  | Shortener  |  |  Metadata    |
        |  Flask :5000 |  | Gin :8000  |  | Express:3000 |
        +------+-------+  +-----+------+  +------+-------+
               |                |                |
            python.db         go.db           node.db
               |                |
               +-------+--------+
                       |
                 +-----v------+
                 |   Redis    |   Pub/Sub (click_events) + cache
                 |   :6379    |   PVC-backed persistence
                 +------------+
```

### Services

| Service | Port | Framework | Responsibility |
|---|---|---|---|
| **Go Service** | 8000 | Gin | Create short codes, fast redirects, Redis cache + publish click events |
| **Python Service** | 5000 | Flask | Web dashboard, orchestrates Go + Node calls, subscribes to Redis click events, analytics |
| **Node.js Service** | 3000 | Express | Fetch page title/description/favicon (Axios + Cheerio), serve metadata REST API |
| **Redis** | 6379 | redis:8-alpine | Message broker (Pub/Sub) and cache layer |

### Communication patterns

**URL creation (synchronous):**

```
User → Python Dashboard
         ├→ Go Service    → create short URL → go.db
         └→ Node Service  → fetch metadata   → node.db
         ↓
    Display short URL + metadata
```

**Click events (event-driven):**

```
User clicks → Go Service
                1. Check Redis cache (hit = instant redirect, miss = DB → cache)
                2. Publish to Redis channel "click_events"
                        ↓
                Python subscriber → process event → python.db
```

- **Python → Go / Node**: HTTP (needs immediate response)
- **Go → Redis → Python**: Pub/Sub (decoupled, async)
- **Fallback**: direct HTTP POST to Python if Redis is unavailable (graceful degradation)

---

## Technology Stack

| Technology | Purpose |
|---|---|
| Docker | Containerization |
| Docker Compose | Local multi-service development |
| Kubernetes | Container orchestration |
| Minikube | Local Kubernetes cluster |
| Redis 8 | Cache + Pub/Sub message broker |
| ConfigMap | Non-sensitive application configuration |
| Secret | Sensitive configuration |
| Ingress NGINX | HTTP routing to services |
| Metrics Server | Resource metrics for HPA |
| Horizontal Pod Autoscaler | CPU-based auto scaling |
| Prometheus | Metrics collection |
| Grafana | Dashboards / visualization |
| k6 | Load and performance testing |
| GitHub Actions | CI pipeline |

---

## Repository Layout

```
urlshortner-microservices/
├── README.md
├── docker-compose.yml              # 4 services: redis, go, node, python
├── loadtest.js                     # k6 load test script
├── .github/
│   └── workflows/
│       └── ci.yml                  # GitHub Actions CI (builds all 3 images)
├── go-service/
│   ├── Dockerfile
│   ├── main.go                     # Gin app, Redis cache + publisher
│   ├── go.mod / go.sum
│   └── .dockerignore
├── node-service/
│   ├── Dockerfile
│   ├── server.js                   # Express metadata API
│   ├── package.json
│   └── .dockerignore
├── python-service/
│   ├── Dockerfile
│   ├── app.py                      # Flask dashboard + Redis subscriber
│   ├── requirements.txt
│   ├── .flake8
│   ├── templates/dashboard.html
│   └── .dockerignore
└── kubernetes/
    ├── namespace.yaml              # namespace: urlshortener
    ├── configmap.yaml              # urlshortener-config
    ├── secret.yaml                 # urlshortener-secret
    ├── redis.yaml                  # PVC + Deployment + Service
    ├── go-deployment.yaml          # Deployment (2 replicas) + ClusterIP Service
    ├── node-deployment.yaml        # Deployment + ClusterIP Service
    ├── python-deployment.yaml      # Deployment + ClusterIP Service
    ├── ingress.yaml                # urlshortener.local → /go /node /python
    └── hpa.yaml                    # 3 HPAs (min 1, max 5, CPU 50%)
```

---

## Prerequisites

Install these before starting:

| Tool | Minimum version | Check |
|---|---|---|
| Docker Desktop | 24+ | `docker --version` |
| Docker Compose | v2 | `docker compose version` |
| Minikube | 1.32+ | `minikube version` |
| kubectl | 1.28+ | `kubectl version --client` |
| Helm | 3.x | `helm version` |
| k6 | 0.49+ | `k6 version` |
| Git | any | `git --version` |

Clone the repository:

```bash
git clone https://github.com/<your-username>/urlshortner-microservices.git
cd urlshortner-microservices
```

> **Note for Windows users:** all commands below work in PowerShell. Where a command uses `\` for line continuation, use a single line or backtick `` ` `` instead. Multi-line `curl` examples use `curl.exe` on Windows.

---

## Deployment — Part 1: Docker Compose (local)

This is the fastest way to run everything and also the step that **builds the images Kubernetes will use**.

### 1.1 Build and start

```bash
docker compose up --build -d
```

### 1.2 Verify

```bash
docker compose ps
```

Expected: `redis`, `go-service`, `node-service`, `python-service` all in `running` state.

```bash
docker compose logs -f            # all services
docker compose logs -f python-service   # one service
```

### 1.3 Access the application

| Component | URL |
|---|---|
| Dashboard (Python) | http://localhost:5000 |
| Go API | http://localhost:8000 |
| Node API | http://localhost:3000 |

Smoke test:

```bash
# Node health
curl http://localhost:3000/health
# → {"status":"healthy"}

# Create a short URL through the Go API
curl -X POST http://localhost:8000/api/shorten \
  -H "Content-Type: application/json" \
  -d '{"long_url":"https://github.com"}'

# Create through the dashboard (orchestrates Go + Node)
curl -X POST http://localhost:5000/create -d "long_url=https://github.com"
```

Open http://localhost:5000 in a browser — the dashboard shows total URLs, total clicks, page titles/favicons fetched by the Node service, and a clicks-over-time chart that refreshes every 5 seconds.

### 1.4 Image names produced

Compose uses the project name `urlshortner-microservices`, so the built images are:

```
urlshortner-microservices-go-service:latest
urlshortner-microservices-node-service:latest
urlshortner-microservices-python-service:latest
```

These are exactly the image names referenced in the Kubernetes deployments. **Do not rename them** unless you also update `kubernetes/*-deployment.yaml`.

### 1.5 Stop

```bash
docker compose down        # stop containers
docker compose down -v     # stop and delete volumes (wipes the databases)
```

---

## Deployment — Part 2: Kubernetes on Minikube

### 2.1 Start the cluster

```bash
minikube start --driver=docker --cpus=4 --memory=6144
minikube status
```

Enable the required addons:

```bash
minikube addons enable ingress          # NGINX Ingress Controller
minikube addons enable metrics-server   # required for HPA
```

Confirm they came up:

```bash
kubectl get pods -n ingress-nginx
kubectl get deployment metrics-server -n kube-system
```

### 2.2 Load the local images into Minikube

The deployments use `imagePullPolicy: Never`, so images must exist **inside** the Minikube node. Build them first with Compose (Part 1), then load:

```bash
minikube image load urlshortner-microservices-go-service:latest
minikube image load urlshortner-microservices-node-service:latest
minikube image load urlshortner-microservices-python-service:latest
```

Verify:

```bash
minikube image ls | grep urlshortner
```

> **Alternative:** build directly inside the Minikube daemon instead of loading.
> Linux/macOS: `eval $(minikube docker-env)` then `docker compose build`.
> PowerShell: `& minikube -p minikube docker-env --shell powershell | Invoke-Expression` then `docker compose build`.

### 2.3 Apply the manifests

Apply in order (namespace first, then config, then workloads):

```bash
kubectl apply -f kubernetes/namespace.yaml
kubectl apply -f kubernetes/configmap.yaml
kubectl apply -f kubernetes/secret.yaml
kubectl apply -f kubernetes/redis.yaml
kubectl apply -f kubernetes/go-deployment.yaml
kubectl apply -f kubernetes/node-deployment.yaml
kubectl apply -f kubernetes/python-deployment.yaml
kubectl apply -f kubernetes/ingress.yaml
kubectl apply -f kubernetes/hpa.yaml
```

Or apply the whole folder at once:

```bash
kubectl apply -f kubernetes/
```

### 2.4 Verify the deployment

```bash
kubectl get all -n urlshortener
kubectl get pods -n urlshortener
kubectl get svc -n urlshortener
```

Wait until every pod is `Running` and `READY 1/1`:

```bash
kubectl wait --for=condition=ready pod --all -n urlshortener --timeout=180s
```

Expected resources:

| Kind | Name | Notes |
|---|---|---|
| Deployment | `go-service` | 2 replicas |
| Deployment | `node-service` | 1 replica |
| Deployment | `python-service` | 1 replica |
| Deployment | `redis` | 1 replica, PVC-backed |
| Service (ClusterIP) | `go-service` | port 8000 |
| Service (ClusterIP) | `node-service` | port 3000 |
| Service (ClusterIP) | `python-service` | port 5000 |
| Service (ClusterIP) | `redis-service` | port 6379 |
| PVC | `redis-pvc` | 1Gi |

### 2.5 Configuration: ConfigMap and Secret

Non-sensitive configuration lives in the ConfigMap `urlshortener-config`:

```bash
kubectl get configmap urlshortener-config -n urlshortener -o yaml
```

| Key | Value |
|---|---|
| `GO_SERVICE_URL` | `http://go-service:8000` |
| `NODE_SERVICE_URL` | `http://node-service:3000` |
| `PYTHON_SERVICE_URL` | `http://python-service:5000` |
| `REDIS_HOST` | `redis-service` |
| `REDIS_PORT` | `6379` |
| `APP_ENV` | `production` |

Sensitive values live in the Secret `urlshortener-secret` (`REDIS_PASSWORD`), kept out of the ConfigMap and out of the container images:

```bash
kubectl get secret urlshortener-secret -n urlshortener
kubectl describe secret urlshortener-secret -n urlshortener
```

> The Secret is declared and available for injection. Redis in this cluster currently runs without `requirepass`, so the deployments read Redis connection details from plain env vars / the ConfigMap. To wire the Secret in, add to a container spec:
>
> ```yaml
> env:
>   - name: REDIS_PASSWORD
>     valueFrom:
>       secretKeyRef:
>         name: urlshortener-secret
>         key: REDIS_PASSWORD
> ```
>
> and start Redis with `--requirepass $(REDIS_PASSWORD)`. Also change the placeholder value before using this anywhere real.

### 2.6 Quick in-cluster test (before Ingress)

```bash
kubectl port-forward -n urlshortener svc/python-service 5000:5000
# browser → http://localhost:5000
```

---

## Deployment — Part 3: Ingress

The Ingress `urlshortener-ingress` routes one host to all three services using path prefixes and a rewrite rule (`rewrite-target: /$2`), so `/node/health` reaches the Node service as `/health`.

| Path | Backend | Port |
|---|---|---|
| `/go/...` | `go-service` | 8000 |
| `/node/...` | `node-service` | 3000 |
| `/python/...` | `python-service` | 5000 |

### 3.1 Verify the Ingress

```bash
kubectl get ingress -n urlshortener
kubectl describe ingress urlshortener-ingress -n urlshortener
```

### 3.2 Map the hostname

Get the Minikube IP:

```bash
minikube ip     # e.g. 192.168.49.2
```

Add a hosts entry:

- **Linux/macOS** — `/etc/hosts` (needs sudo):
  ```
  192.168.49.2   urlshortener.local
  ```
- **Windows** — `C:\Windows\System32\drivers\etc\hosts` (open Notepad as Administrator):
  ```
  127.0.0.1   urlshortener.local
  ```

### 3.3 Reach the Ingress

On Linux/macOS the Minikube IP is routable directly:

```bash
curl http://urlshortener.local/node/health
```

On Windows (Docker driver) the node IP is not routable from the host, so port-forward the ingress controller to **port 8080** — this is the setup the k6 test expects:

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

Then in another terminal:

```bash
curl -H "Host: urlshortener.local" http://localhost:8080/node/health
# → {"status":"healthy"}

curl -H "Host: urlshortener.local" http://localhost:8080/python/
```

Browser access: add `127.0.0.1 urlshortener.local` to your hosts file and open
`http://urlshortener.local:8080/python/`.

---

## Deployment — Part 4: Horizontal Pod Autoscaler

Three HPAs are configured — one per application service.

| HPA | Target Deployment | Min | Max | CPU target | Scale-down window |
|---|---|---|---|---|---|
| `go-service-hpa` | `go-service` | 1 | 5 | 50% | 60s |
| `node-service-hpa` | `node-service` | 1 | 5 | 50% | 60s |
| `python-service-hpa` | `python-service` | 1 | 5 | 50% | 60s |

HPA requires the Metrics Server (enabled in step 2.1) and CPU **requests** on the containers — every deployment in this project sets `requests.cpu: 100m`.

### 4.1 Apply and verify

```bash
kubectl apply -f kubernetes/hpa.yaml
kubectl get hpa -n urlshortener
```

The `TARGETS` column must show a real percentage (e.g. `1%/50%`). If it shows `<unknown>`, the Metrics Server is not ready yet — wait ~60 seconds and re-check:

```bash
kubectl top pods -n urlshortener
```

### 4.2 Watch scaling during load

Run this in a separate terminal while the k6 test (Part 6) is running:

```bash
kubectl get hpa -n urlshortener -w
kubectl get pods -n urlshortener -w
```

---

## Deployment — Part 5: Monitoring with Prometheus & Grafana

The `kube-prometheus-stack` Helm chart installs Prometheus, Grafana, Alertmanager and the node/kube-state exporters in one shot.

### 5.1 Install

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```

PowerShell single-line version:

```powershell
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring
```

### 5.2 Verify

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Wait for all pods to reach `Running`:

```bash
kubectl wait --for=condition=ready pod --all -n monitoring --timeout=300s
```

### 5.3 Open Grafana

```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3001:80
```

Open http://localhost:3001

Credentials:

- **Username:** `admin`
- **Password:** retrieve it with

  ```bash
  kubectl get secret monitoring-grafana -n monitoring \
    -o jsonpath="{.data.admin-password}" | base64 -d
  ```

  PowerShell:

  ```powershell
  $b = kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}"
  [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($b))
  ```

  (Chart default is `prom-operator`.)

Useful built-in dashboards: **Kubernetes / Compute Resources / Namespace (Pods)** — select namespace `urlshortener` to watch CPU and memory per pod while load testing.

### 5.4 Open Prometheus

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
```

Open http://localhost:9090 and try:

```promql
sum(rate(container_cpu_usage_seconds_total{namespace="urlshortener"}[2m])) by (pod)
container_memory_working_set_bytes{namespace="urlshortener"}
kube_deployment_status_replicas{namespace="urlshortener"}
```

---

## Deployment — Part 6: Load Testing with k6

[`loadtest.js`](loadtest.js) drives traffic through the Ingress at `localhost:8080` with the `Host: urlshortener.local` header.

```javascript
export const options = {
  vus: 50,          // 50 virtual users
  duration: '30s',  // for 30 seconds
};
```

### 6.1 Prerequisites

The ingress port-forward from step 3.3 must be running:

```bash
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

### 6.2 Install k6

- **Windows:** `winget install k6 --source winget` or `choco install k6`
- **macOS:** `brew install k6`
- **Linux:** see https://grafana.com/docs/k6/latest/set-up/install-k6/

Verify: `k6 version` (if "command not found", reopen the terminal so the PATH refreshes).

### 6.3 Run the test

```bash
k6 run loadtest.js
```

### 6.4 Results achieved

| Metric | Result |
|---|---|
| Virtual users | 50 |
| Duration | 30s |
| Total requests | 1500 |
| Failed requests | 0 (0.00%) |
| Average response time | ≈ 10–12 ms |

The application served all concurrent requests successfully with no failures.

### 6.5 Observe autoscaling during the run

```bash
# terminal A
k6 run loadtest.js

# terminal B
kubectl get hpa -n urlshortener -w

# terminal C
kubectl top pods -n urlshortener
```

> The default script only hits the lightweight `/node/health` endpoint, so CPU stays low and the HPA may not scale. To force scaling, raise `vus` (e.g. 300), remove the `sleep(1)`, or point the request at `/python/` which does more work per request.

---

## CI/CD with GitHub Actions

Workflow: [.github/workflows/ci.yml](.github/workflows/ci.yml)

**Triggers:** every push to `main` and every pull request targeting `main`.

**Job — `build` (ubuntu-latest):**

1. `actions/checkout@v4`
2. `docker build -t go-service ./go-service`
3. `docker build -t node-service ./node-service`
4. `docker build -t python-service ./python-service`

This validates that all three Dockerfiles build cleanly before any change is merged — a broken Dockerfile fails the pipeline instead of the deployment.

Check runs under the repository's **Actions** tab.

### Extending to CD

To push images and deploy automatically, add a job after `build`:

```yaml
  push:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - run: |
          docker build -t $DOCKERHUB_USERNAME/go-service:${{ github.sha }} ./go-service
          docker push $DOCKERHUB_USERNAME/go-service:${{ github.sha }}
```

Then swap `imagePullPolicy: Never` for a registry image reference in the deployments and run `kubectl set image` (or Argo CD / Flux) against the target cluster.

---

## Verification Checklist

Run these to confirm every part of the assignment:

```bash
# Docker
docker compose ps

# Namespace and workloads
kubectl get all -n urlshortener

# Pods
kubectl get pods -n urlshortener

# Services
kubectl get svc -n urlshortener

# ConfigMap and Secret
kubectl get configmap,secret -n urlshortener

# Ingress
kubectl get ingress -n urlshortener

# HPA
kubectl get hpa -n urlshortener

# Metrics
kubectl top pods -n urlshortener

# Monitoring stack
kubectl get pods -n monitoring

# Application through the Ingress
curl -H "Host: urlshortener.local" http://localhost:8080/node/health

# Load test
k6 run loadtest.js
```

---

## Cleanup / Teardown

```bash
# Application resources
kubectl delete -f kubernetes/
# or drop the whole namespace (removes everything inside it)
kubectl delete namespace urlshortener

# Monitoring stack
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring

# Docker Compose
docker compose down -v

# Whole cluster
minikube stop
minikube delete
```

---

## Troubleshooting

Issues encountered during this assignment and how each was resolved.

**`ErrImageNeverPull` / `ImagePullBackOff`**
The image is not present inside the Minikube node. Build with `docker compose build`, then `minikube image load <image>:latest` for all three images. Confirm with `minikube image ls | grep urlshortner`. The image name must match the deployment exactly, including the `urlshortner-microservices-` prefix.

**Ingress not reachable / connection refused**
`minikube addons enable ingress` and wait for `kubectl get pods -n ingress-nginx` to show `Running`. On Windows with the Docker driver the Minikube IP is not routable from the host — use `kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80` and send the `Host: urlshortener.local` header.

**404 from the Ingress**
The rewrite annotation strips the prefix, so request `/node/health`, not `/node` alone. Requests without the correct `Host` header will not match the rule.

**`HPA TARGETS <unknown>`**
Metrics Server is missing or still warming up: `minikube addons enable metrics-server`, wait ~60s, then check `kubectl top pods -n urlshortener`. HPA also needs CPU `requests` set on the containers (already configured here).

**Helm install fails / chart not found**
Run `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts` followed by `helm repo update` before installing.

**Grafana pod stuck in `Init` / login fails**
The init container waits on the datasource sidecar — give it 2–3 minutes. Fetch the real password from the `monitoring-grafana` secret rather than assuming the default.

**`k6: command not found` after install**
The installer updated PATH but the current shell has the old environment. Close and reopen the terminal, or run k6 from its install directory.

**Pod `CrashLoopBackOff`**
Read the logs and events:

```bash
kubectl logs -n urlshortener <pod-name>
kubectl logs -n urlshortener <pod-name> --previous
kubectl describe pod -n urlshortener <pod-name>
```

**Redis PVC `Pending`**
Minikube's default storage class must be active: `kubectl get sc`. Check the claim with `kubectl describe pvc redis-pvc -n urlshortener`.

---

## API Reference

### Go Service — port 8000

**Create short URL**

```http
POST /api/shorten
Content-Type: application/json

{ "long_url": "https://example.com/very/long/url" }
```

```json
{
  "short_code": "abc123",
  "short_url": "http://localhost:8000/abc123",
  "long_url": "https://example.com/very/long/url"
}
```

**Redirect**

```http
GET /{short_code}
```

Redirects to the long URL and publishes a click event to Redis.

### Python Service — port 5000

| Endpoint | Method | Description |
|---|---|---|
| `/` | GET | Web dashboard |
| `/create` | POST | Create a URL from the UI (`long_url` form field) |
| `/api/events` | POST | HTTP fallback for click events |
| `/api/stats` | GET | `total_urls`, `total_clicks`, `top_urls`, `recent_clicks`, `clicks_over_time`, `all_urls` |

### Node.js Service — port 3000

**Fetch metadata**

```http
POST /api/metadata
Content-Type: application/json

{ "short_code": "abc123", "long_url": "https://example.com" }
```

```json
{
  "short_code": "abc123",
  "url": "https://example.com",
  "title": "Example Domain",
  "description": "Example domain for documentation",
  "favicon_url": "https://example.com/favicon.ico",
  "status": "success"
}
```

| Endpoint | Method | Description |
|---|---|---|
| `/api/metadata/{short_code}` | GET | Stored metadata for a short code |
| `/health` | GET | `{"status":"healthy"}` |

---

## Database Schema

**Go — `go.db`**

```sql
CREATE TABLE urls (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    short_code TEXT UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

**Python — `python.db`**

```sql
CREATE TABLE click_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    short_code TEXT NOT NULL,
    clicked_at DATETIME NOT NULL
);

CREATE TABLE url_metadata (
    short_code TEXT PRIMARY KEY,
    long_url TEXT NOT NULL,
    total_clicks INTEGER DEFAULT 0,
    first_seen DATETIME NOT NULL,
    last_clicked DATETIME,
    title TEXT,
    description TEXT,
    favicon_url TEXT,
    metadata_status TEXT DEFAULT 'pending'
);
```

**Node.js — `node.db`**

```sql
CREATE TABLE metadata (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    short_code TEXT UNIQUE NOT NULL,
    url TEXT NOT NULL,
    title TEXT,
    description TEXT,
    favicon_url TEXT,
    fetched_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## Learning Outcomes

- Containerizing polyglot services (Go, Python, Node.js) with per-service Dockerfiles
- Orchestrating a multi-service stack locally with Docker Compose
- Writing Kubernetes Deployments, Services, PVCs and Namespaces from scratch
- Separating configuration (ConfigMap) from secrets (Secret)
- Routing multiple services behind a single host with Ingress NGINX and rewrite rules
- Configuring CPU-based autoscaling with HPA and the Metrics Server
- Installing and using a monitoring stack (Prometheus + Grafana) via Helm
- Performance testing with k6 and correlating results with cluster metrics
- Automating build validation with GitHub Actions CI

---

## Credits & License

Base application originally authored by [Abdullah Zayed](https://zayedabdullah.com) ([GitHub: xaadu](https://github.com/xaadu)).

Kubernetes deployment, Ingress, HPA, monitoring, load testing and CI/CD work by **Mahbuba Sultana Shammi** — Ostad Batch-10, DevOps.

MIT License — free to use for educational purposes.
