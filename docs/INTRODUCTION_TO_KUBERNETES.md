# Document 2 — Introduction to Kubernetes

**Phase 4 — Kubernetes on Raspberry Pi**  
**Version:** 1.0  

---

## Overview

This document explains what Kubernetes is, why it matters, and how it differs from Docker Compose. It covers the core concepts and architecture used throughout Phase 4 before any installation begins.

---

## 2.1 What is Kubernetes?

Docker Compose (Phase 3) ran all containers on a single machine. Kubernetes runs containers across multiple machines and manages them automatically.

```
Docker Compose (Phase 3):          Kubernetes (Phase 4):
Single Machine                     Multiple Machines (Cluster)
├── Frontend Container             ├── Raspberry Pi #1 (Control Plane)
├── Backend Container              └── Raspberry Pi #2 (Worker)
├── PostgreSQL Container               Manages all containers across both:
└── Redis Container                    ├── Automatic restart on failure
                                       ├── Load balancing
                                       ├── Rolling updates (zero downtime)
                                       └── Persistent storage
```

---

## 2.2 Why Kubernetes Matters

| Without Kubernetes | With Kubernetes |
|-------------------|-----------------|
| Container crashes → manual restart | Container crashes → automatic restart |
| High load → manually add containers | High load → automatic scaling |
| Updates → downtime | Updates → zero-downtime rolling updates |
| Monitoring → stare at logs | Monitoring → automated alerting |

---

## 2.3 Cluster Architecture

```
HOME NETWORK (192.168.30.0/24)

┌─────────────────────┐        ┌──────────────────────┐
│  cp-01              │        │  wrk-01              │
│  192.168.30.20      │◄──────►│  192.168.30.21       │
├─────────────────────┤        ├──────────────────────┤
│ Control Plane       │        │ Worker Node          │
│ ✓ API Server        │        │ ✓ kubelet            │
│ ✓ etcd              │        │ ✓ containerd         │
│ ✓ Scheduler         │        │ ✓ kube-proxy         │
│ ✓ Controller Mgr    │        │                      │
└─────────────────────┘        └──────────────────────┘
           ▲                              ▲
           └──────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              │  sre-mgmnt-01         │
              │  kubectl access       │
              │  GitOps management    │
              └───────────────────────┘
```

**Control Plane (cp-01):**
- **API Server** — receives `kubectl` commands from `sre-mgmnt-01`
- **etcd** — stores all cluster state (pod definitions, secrets, volumes)
- **Scheduler** — decides which node new pods should run on
- **Controller Manager** — ensures desired state matches actual state ("you want 2 replicas running? Only 1 is. Starting another now.")

**Worker Node (wrk-01):**
- **kubelet** — agent that receives instructions from the control plane
- **containerd** — actually runs the containers (lighter than Docker: 150MB vs 500MB)
- **kube-proxy** — routes network traffic between pods

---

## 2.4 Key Kubernetes Concepts

**Pod** — the smallest deployable unit in Kubernetes. Usually one container per pod. The pod wraps the container with Kubernetes management.

**Deployment** — defines how many copies of a pod to run and how to update them.

```yaml
spec:
  replicas: 2    # Run 2 copies of this pod
```

**Service** — a stable network endpoint. Pods restart and change IP addresses; a Service stays constant and load-balances across them.

**ConfigMap** — stores non-sensitive configuration such as environment variables and settings.

**Secret** — stores sensitive data such as passwords and API keys. Kubernetes base64-encodes and restricts access to Secrets.

**PersistentVolumeClaim** — requests storage that outlives the pod. When a pod restarts, it reconnects to the same data.

**Ingress** — routes external HTTP/HTTPS traffic into the cluster.

---

## 2.5 Hardware Used in Phase 4

| Component | Quantity | Spec |
| ---------- | -------- | ---- |
| Raspberry Pi 5 | 1 | 8GB RAM — Control Plane (`cp-01`) |
| Raspberry Pi 4 | 1 | 4GB RAM — Worker Node (`wrk-01`) |
| microSD or USB SSD | 2 | 64GB minimum |
| Ethernet cables | 2 | Cat6, wired — WiFi not recommended |
| Power supplies | 2 | 5V 3A USB-C |

> **Storage note:** USB SSD is strongly recommended over microSD. Kubernetes does heavy I/O. microSD cards wear out faster and are significantly slower (90MB/s vs 450MB/s).

---

## 2.6 Node Reference

| Node | Role | IP | User | Hardware |
|------|------|----|------|----------|
| `cp-01` | Control Plane | 192.168.30.20 | kubeadmin | Raspberry Pi 5 |
| `wrk-01` | Worker Node | 192.168.30.21 | kubeadmin | Raspberry Pi 4 |
| `sre-mgmnt-01` | Management VM | 192.168.30.12 | sre | Arch Linux VM |

---

## 2.7 Network Reference

| VLAN | Name | Subnet |
|------|------|--------|
| 1 | Default | 192.168.0.0/24 |
| 30 | Home Lab | 192.168.30.0/24 |

All cluster nodes and the management VM sit on VLAN 30 (Home Lab).

---

## 2.8 From Docker Compose to Kubernetes Manifests

In Phase 3 (Docker), the entire application stack was defined in a single `docker-compose.yml` file. In Phase 4 (Kubernetes), each concern is broken out into a separate YAML manifest file. The concepts map directly — the terminology and structure change, but the intent is the same.

This section shows how each Docker Compose service definition translates into its Kubernetes equivalent, using the PostgreSQL and backend services as worked examples.

---

### Concept Mapping

| Docker Compose | Kubernetes | Purpose |
|----------------|------------|---------|
| `image:` | `spec.containers[].image` in a Deployment | Which container image to run |
| `environment:` (non-sensitive) | ConfigMap injected via `envFrom` | Runtime configuration |
| `environment:` (sensitive) | Secret injected via `envFrom` | Passwords, keys |
| `.env` file | ConfigMap + Secrets | Separates safe and sensitive config |
| `ports:` (host mapping) | NodePort Service | Exposes a port externally |
| `ports:` (internal only) | ClusterIP Service | Internal pod-to-pod communication |
| `volumes:` named volume | PersistentVolumeClaim | Storage that outlives the container |
| `networks:` | Kubernetes namespace + Services | Isolation and DNS-based discovery |
| `depends_on:` | `livenessProbe` / `readinessProbe` | Startup ordering and health checking |
| `restart: unless-stopped` | Built into Kubernetes | Automatic — no configuration needed |

---

### Example 1 — PostgreSQL

**Docker Compose (Phase 3):**

```yaml
postgres:
  image: postgres:15.2-alpine
  restart: unless-stopped
  environment:
    POSTGRES_DB: ${DB_NAME:-humor_memory_game}
    POSTGRES_USER: ${DB_USER:-gameuser}
    POSTGRES_PASSWORD: ${DB_PASSWORD:-gamepass123}
  volumes:
    - postgres_data:/var/lib/postgresql/data
  networks:
    - backend-network
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-gameuser} -d ${DB_NAME:-humor_memory_game}"]
    interval: 10s
    timeout: 5s
    retries: 5
    start_period: 30s
```

**Kubernetes equivalent — four separate files:**

`image:` and `healthcheck:` become part of the **Deployment**:

```yaml
# 06-postgres-deployment.yaml (excerpt)
containers:
- name: postgres
  image: postgres:15-alpine          # image: maps directly
  livenessProbe:                     # healthcheck: becomes livenessProbe
    exec:
      command:
      - /bin/sh
      - -c
      - pg_isready -U gameuser -d humor_memory_game
    initialDelaySeconds: 30          # start_period: equivalent
    periodSeconds: 10                # interval: equivalent
```

`environment:` non-sensitive values move to the **ConfigMap**:

```yaml
# 03-configmap.yaml (excerpt)
data:
  DB_NAME: humor_memory_game         # POSTGRES_DB source value
```

`environment:` sensitive values move to a **Secret** (created with kubectl, never stored in Git):

```bash
kubectl create secret generic database-secret \
  --from-literal=DB_USER=gameuser \
  --from-literal=DB_PASSWORD=gamepass123 \
  -n humor-memory-game
```

`volumes:` named volume becomes a **PersistentVolumeClaim**:

```yaml
# 04-pvc-postgres.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce                  # One node at a time — same as a named volume
  resources:
    requests:
      storage: 5Gi                   # Explicit size — Docker named volumes grow freely
  storageClassName: synology-iscsi-storage
```

`networks: backend-network` (internal only) becomes a **ClusterIP Service**:

```yaml
# 07-postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres                     # This name replaces the Docker network DNS entry
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP                    # Internal only — same intent as backend-network
```

> In Docker Compose, `DB_HOST: postgres` worked because Docker DNS resolved the service name `postgres` on the backend network. In Kubernetes, the same value works because CoreDNS resolves the Service name `postgres` within the namespace. The application code does not change.

`restart: unless-stopped` requires no configuration in Kubernetes — pods are automatically restarted by the cluster if they crash.

---

### Example 2 — Backend

**Docker Compose (Phase 3):**

```yaml
backend:
  build:
    context: ../humor-memory-game/backend/
    dockerfile: /home/devops/workspace/humor-memory-game-devops/docker/backend/Dockerfile
  image: humor-memory-game-backend:latest
  restart: unless-stopped
  environment:
    NODE_ENV: ${NODE_ENV:-development}
    DB_HOST: postgres
    DB_PORT: 5432
    DB_NAME: ${DB_NAME:-humor_memory_game}
    DB_USER: ${DB_USER:-gameuser}
    DB_PASSWORD: ${DB_PASSWORD:-gamepass123}
    REDIS_HOST: redis
    REDIS_PORT: 6379
    REDIS_PASSWORD: ${REDIS_PASSWORD:-gamepass123}
    JWT_SECRET: ${JWT_SECRET:-dev-secret-key-change-in-production}
    API_BASE_URL: ${API_BASE_URL:-/api}
    CORS_ORIGIN: ${CORS_ORIGIN:-http://frontend:80}
  ports:
    - "3001:3001"
  networks:
    - backend-network
    - frontend-network
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_healthy
```

**Kubernetes equivalent:**

`build:` is gone entirely — Kubernetes does not build images. Images must already be built and pushed to a registry such as Docker Hub before deployment. Because the Raspberry Pi cluster runs ARM64 and the development machine runs AMD64, images must be built for both architectures using `docker buildx` before they can run on the cluster.

`image:` maps directly to the Deployment, now referencing the pre-built Docker Hub image:

```yaml
# 10-backend-deployment.yaml (excerpt)
image: christie62/humor-memory-game-backend:latest
imagePullPolicy: Always              # Always pull from Docker Hub — not a local build
```

`environment:` is replaced by `envFrom` — pulling all values from the ConfigMap and Secrets in a single block rather than listing each variable individually:

```yaml
# 10-backend-deployment.yaml (excerpt)
envFrom:
- configMapRef:
    name: app-config                 # Injects NODE_ENV, DB_HOST, DB_PORT, DB_NAME, etc.
- secretRef:
    name: database-secret            # Injects DB_USER, DB_PASSWORD
- secretRef:
    name: redis-secret               # Injects REDIS_PASSWORD
- secretRef:
    name: app-secret                 # Injects JWT_SECRET, SESSION_SECRET
```

`ports: "3001:3001"` (internal between services) becomes a **ClusterIP Service**:

```yaml
# 11-backend-service.yaml
spec:
  ports:
  - port: 3001
    targetPort: 3001
  type: ClusterIP                    # Frontend reaches backend via 'backend:3001'
```

`depends_on: service_healthy` becomes `readinessProbe` — Kubernetes will not route traffic to a pod until the readiness probe passes:

```yaml
# 10-backend-deployment.yaml (excerpt)
readinessProbe:
  httpGet:
    path: /api/health
    port: 3001
  initialDelaySeconds: 5            # Wait before first check — equivalent to depends_on delay
  periodSeconds: 5
```

`networks: backend-network + frontend-network` — the backend was on both Docker networks to bridge frontend and database traffic. In Kubernetes this is not needed. All pods in the same namespace can communicate via Services. The namespace itself provides the isolation.

---

### Key Differences to Note

**Explicit image builds.** Docker Compose can build images on the fly using `build:`. Kubernetes cannot — images must be pre-built and pushed to a registry such as Docker Hub before deployment. Because the Raspberry Pi cluster runs ARM64 and the development machine runs AMD64, images must be built for both architectures using `docker buildx` before they can run on the cluster.

**Explicit storage sizing.** Docker named volumes grow freely. Kubernetes PersistentVolumeClaims require an explicit size upfront — `storage: 5Gi` for PostgreSQL, `storage: 2Gi` for Redis.

**Configuration is split.** Docker Compose reads a single `.env` file. Kubernetes separates non-sensitive values (ConfigMap) from sensitive values (Secrets). The ConfigMap is safe to commit to Git. Secrets are never committed — they are created directly with `kubectl create secret`.

**No build-time networking.** Docker Compose networks are defined in the same file as the services. Kubernetes networking is handled automatically by CoreDNS and the namespace — no network definitions are required in the manifests.

---

*Next: Document 3 — Preparing the Raspberry Pis*
