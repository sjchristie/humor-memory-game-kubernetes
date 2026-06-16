# Document 5 — Deploying the Application

**Phase 4 — Kubernetes on Raspberry Pi**  
**Version:** 1.0  

---

## Overview

This document covers building multi-platform Docker images, creating all Kubernetes manifest files, deploying the full application stack to the cluster, and verifying the application is running end-to-end.

> **Prerequisites:** The cluster must be fully operational and `kubectl` configured on `sre-mgmnt-01` as per Document 4 before starting here.

---

## 5.1 Understanding the Conversion from Docker Compose

In Phase 3 (Docker) you had one `docker-compose.yml` file with all four services. Kubernetes uses separate manifest files for each concern. This separation is deliberate — it makes individual components reusable, version-controllable, and independently manageable.

**Docker Compose (Phase 3):**

```
docker-compose.yml
└── services: postgres, redis, backend, frontend
```

**Kubernetes (Phase 4):**

```
manifests/
├── 01-namespace.yaml
├── 02-secrets.yaml          # Reference only — never committed to Git
├── 03-configmap.yaml
├── 04-pvc-postgres.yaml
├── 05-pvc-redis.yaml
├── 06-postgres-deployment.yaml
├── 07-postgres-service.yaml
├── 08-redis-deployment.yaml
├── 09-redis-service.yaml
├── 10-backend-deployment.yaml
├── 11-backend-service.yaml
├── 12-frontend-deployment.yaml
├── 13-frontend-service.yaml
└── 14-ingress.yaml          # Reference only — not applied in Phase 4
```

---

## 5.2 Understanding Configuration Management

In Docker Compose, configuration came from a `.env` file. In Kubernetes it is split between ConfigMaps and Secrets.

**ConfigMap** — non-sensitive values that can be stored in Git:

| Variable | Value | Why |
|----------|-------|-----|
| `DB_HOST` | `postgres` | Kubernetes DNS resolves the Service name automatically |
| `DB_PORT` | `5432` | Standard PostgreSQL port |
| `DB_NAME` | `humor_memory_game` | Database name |
| `REDIS_HOST` | `redis` | Kubernetes DNS resolves the Service name automatically |
| `REDIS_PORT` | `6379` | Standard Redis port |
| `NODE_ENV` | `development` | Runtime environment |
| `API_PORT` | `3001` | Backend listens on this port |
| `API_BASE_URL` | `/api` | Frontend API routing prefix |

> `DB_HOST: postgres` and `REDIS_HOST: redis` are the names of Kubernetes **Services**, not hostnames. CoreDNS (K3s's built-in DNS) automatically resolves `postgres` to the IP of the postgres Service within the cluster. This is how microservices find each other — no hardcoded IPs.

**Secrets** — sensitive values that must never go into Git:

| Secret Name | Keys |
|-------------|------|
| `database-secret` | `DB_USER`, `DB_PASSWORD` |
| `redis-secret` | `REDIS_PASSWORD` |
| `app-secret` | `JWT_SECRET`, `SESSION_SECRET` |

---

## 5.3 Create the Manifest Directory

All commands in this section are run from `sre-mgmnt-01`:

```bash
ssh sre-mgmnt-01
```

```bash
mkdir -p ~/humor-memory-game-kubernetes/manifests

cd ~/humor-memory-game-kubernetes/manifests
```

---

## 5.4 Create the Manifest Files

### 01-namespace.yaml

The namespace isolates all application resources from other workloads on the cluster.

Annotated Version — Read First

```yaml
# Namespace — humor-memory-game
# Isolates all application resources from other cluster workloads
# All subsequent manifests reference this namespace

apiVersion: v1
kind: Namespace
metadata:
  name: humor-memory-game       # All resources will be created in this namespace
  labels:
    name: humor-memory-game     # Label for selecting or filtering this namespace
```

Production Copy — Run This Command

```bash
cat > 01-namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: humor-memory-game
  labels:
    name: humor-memory-game
EOF
```

Verify:

```bash
cat 01-namespace.yaml
```

---

### 02-secrets.yaml

> ⚠️ Secrets are **never** stored in Git. This file is a reference only — it documents the `kubectl` commands used to create secrets directly in the cluster. See Section 5.6 for the actual secret creation commands.

```bash
cat > 02-secrets.yaml << 'EOF'
# DO NOT commit real secret values to Git.
# Create secrets directly with kubectl:
#
# kubectl create secret generic database-secret \
#   --from-literal=DB_USER=gameuser \
#   --from-literal=DB_PASSWORD=<your-password> \
#   -n humor-memory-game
#
# kubectl create secret generic redis-secret \
#   --from-literal=REDIS_PASSWORD=<your-password> \
#   -n humor-memory-game
#
# kubectl create secret generic app-secret \
#   --from-literal=JWT_SECRET=<your-jwt-secret> \
#   --from-literal=SESSION_SECRET=<your-session-secret> \
#   -n humor-memory-game
EOF
```

---

### 03-configmap.yaml

The ConfigMap stores all non-sensitive runtime configuration for the application.

Annotated Version — Read First

```yaml
# ConfigMap — app-config
# Stores non-sensitive configuration shared across backend and frontend pods
# Safe to commit to Git — contains no passwords or secrets

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: humor-memory-game
data:
  NODE_ENV: development
  DB_HOST: postgres           # Kubernetes Service name — CoreDNS resolves this automatically
  DB_PORT: "5432"             # String format required for env var injection
  DB_NAME: humor_memory_game
  REDIS_HOST: redis           # Kubernetes Service name — CoreDNS resolves this automatically
  REDIS_PORT: "6379"
  API_PORT: "3001"
  API_BASE_URL: /api          # Frontend uses this prefix for all API requests
  CORS_ORIGIN: http://frontend:80   # Backend allows requests from the frontend Service
```

Production Copy — Run This Command

```bash
cat > 03-configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: humor-memory-game
data:
  NODE_ENV: development
  DB_HOST: postgres
  DB_PORT: "5432"
  DB_NAME: humor_memory_game
  REDIS_HOST: redis
  REDIS_PORT: "6379"
  API_PORT: "3001"
  API_BASE_URL: /api
  CORS_ORIGIN: http://frontend:80
EOF
```

Verify:

```bash
cat 03-configmap.yaml
```

---

### 04-pvc-postgres.yaml

The PostgreSQL PersistentVolumeClaim requests 5Gi of iSCSI storage from NAS2 via the Synology CSI driver. This ensures database files survive pod restarts and node rescheduling.

Annotated Version — Read First

```yaml
# PersistentVolumeClaim — postgres-pvc
# Requests 5Gi of storage from the Synology iSCSI storage class
# ReadWriteOnce — only one node can mount this volume at a time (correct for databases)
# Data survives pod restarts and node rescheduling — stored on NAS2, not on the Pi

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: humor-memory-game
spec:
  accessModes:
    - ReadWriteOnce             # One node at a time — required for block storage (iSCSI)
  resources:
    requests:
      storage: 5Gi              # 5GB allocated on NAS2 for PostgreSQL data
  storageClassName: synology-iscsi-storage  # Retain policy — data preserved if PVC deleted
```

Production Copy — Run This Command

```bash
cat > 04-pvc-postgres.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: humor-memory-game
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: synology-iscsi-storage
EOF
```

Verify:

```bash
cat 04-pvc-postgres.yaml
```

---

### 05-pvc-redis.yaml

The Redis PersistentVolumeClaim requests 2Gi of iSCSI storage for Redis append-only file persistence.

Annotated Version — Read First

```yaml
# PersistentVolumeClaim — redis-pvc
# Requests 2Gi of storage from the Synology iSCSI storage class
# Redis uses appendonly mode — all writes are logged to /data on this volume

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: humor-memory-game
spec:
  accessModes:
    - ReadWriteOnce             # One node at a time — required for block storage (iSCSI)
  resources:
    requests:
      storage: 2Gi              # 2GB allocated on NAS2 for Redis persistence
  storageClassName: synology-iscsi-storage  # Retain policy — data preserved if PVC deleted
```

Production Copy — Run This Command

```bash
cat > 05-pvc-redis.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: humor-memory-game
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: synology-iscsi-storage
EOF
```

Verify:

```bash
cat 05-pvc-redis.yaml
```

---

### 06-postgres-deployment.yaml

The PostgreSQL Deployment runs a single replica with credentials injected from ConfigMap and Secrets. The `PGDATA` environment variable is set to a subdirectory of the mount point — this is required because iSCSI LUNs are formatted as ext4 and contain a `lost+found` directory at the root, which PostgreSQL rejects as a non-empty data directory.

Annotated Version — Read First

```yaml
# Deployment — postgres
# Runs PostgreSQL 15 with persistent iSCSI storage
# Credentials injected from ConfigMap (DB_NAME) and Secrets (DB_USER, DB_PASSWORD)
# PGDATA set to subdirectory to avoid lost+found conflict on iSCSI ext4 mount

apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: humor-memory-game
  labels:
    app: postgres
spec:
  replicas: 1                   # Always 1 for databases — multiple replicas require clustering
  selector:
    matchLabels:
      app: postgres             # Must match template labels below
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine   # Alpine variant — smaller image size
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_NAME          # Pulled from ConfigMap — not sensitive
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: DB_USER          # Pulled from Secret — never hardcoded
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: DB_PASSWORD      # Pulled from Secret — never hardcoded
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
          # Subdirectory required — iSCSI ext4 mounts contain lost+found at root
          # PostgreSQL refuses to initialise in a non-empty directory
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data   # PVC mounted here
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U gameuser -d humor_memory_game
          initialDelaySeconds: 30   # Give PostgreSQL time to initialise before first check
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U gameuser -d humor_memory_game
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc   # References the PVC created in 04-pvc-postgres.yaml
```

Production Copy — Run This Command

```bash
cat > 06-postgres-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: humor-memory-game
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_NAME
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: DB_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: DB_PASSWORD
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U gameuser -d humor_memory_game
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - pg_isready -U gameuser -d humor_memory_game
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
EOF
```

Verify:

```bash
cat 06-postgres-deployment.yaml
```

---

### 07-postgres-service.yaml

The PostgreSQL Service exposes the database internally to other pods using a stable ClusterIP. Nothing outside the cluster can reach it.

Annotated Version — Read First

```yaml
# Service — postgres (ClusterIP)
# Exposes PostgreSQL internally within the cluster
# ClusterIP — no external access, reachable only by other pods
# Backend connects to PostgreSQL via 'postgres:5432' — CoreDNS resolves 'postgres' to this Service

apiVersion: v1
kind: Service
metadata:
  name: postgres                # This name is what 'DB_HOST: postgres' in the ConfigMap resolves to
  namespace: humor-memory-game
spec:
  selector:
    app: postgres               # Routes traffic to pods with this label
  ports:
  - port: 5432                  # Port exposed by the Service
    targetPort: 5432            # Port the PostgreSQL container listens on
    protocol: TCP
  type: ClusterIP               # Internal only — not accessible outside the cluster
```

Production Copy — Run This Command

```bash
cat > 07-postgres-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: humor-memory-game
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
    protocol: TCP
  type: ClusterIP
EOF
```

Verify:

```bash
cat 07-postgres-service.yaml
```

---

### 08-redis-deployment.yaml

The Redis Deployment runs a single replica with append-only persistence enabled and password authentication. The image is pinned to `redis:7.2.4-alpine` — see Known Issue in Section 5.9.

Annotated Version — Read First

```yaml
# Deployment — redis
# Runs Redis 7.2.4 with append-only persistence and password authentication
# Image pinned to redis:7.2.4-alpine — floating tags (redis:7-alpine) caused
# architecture cache issues on ARM64 nodes. See Known Issue — Redis Architecture.

apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: humor-memory-game
  labels:
    app: redis
spec:
  replicas: 1                   # Single replica — Redis Cluster requires separate setup
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7.2-alpine
        command:
        - redis-server
        - "--appendonly"
        - "yes"                       # Enables append-only file persistence to /data
        - "--requirepass"
        - "$(REDIS_PASSWORD)"         # Password injected from Secret at runtime
        ports:
        - containerPort: 6379
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: REDIS_PASSWORD     # Pulled from Secret — never hardcoded
        volumeMounts:
        - name: redis-storage
          mountPath: /data            # Redis writes append-only log here
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - redis-cli -a $(REDIS_PASSWORD) ping
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - redis-cli -a $(REDIS_PASSWORD) ping
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc        # References the PVC created in 05-pvc-redis.yaml
```

Production Copy — Run This Command

```bash
cat > 08-redis-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: humor-memory-game
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7.2-alpine
        command:
        - redis-server
        - "--appendonly"
        - "yes"
        - "--requirepass"
        - "$(REDIS_PASSWORD)"
        ports:
        - containerPort: 6379
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: REDIS_PASSWORD
        volumeMounts:
        - name: redis-storage
          mountPath: /data
        livenessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - redis-cli -a $(REDIS_PASSWORD) ping
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - /bin/sh
            - -c
            - redis-cli -a $(REDIS_PASSWORD) ping
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
EOF
```

Verify:

```bash
cat 08-redis-deployment.yaml
```

---

### 09-redis-service.yaml

The Redis Service exposes the cache internally to the backend pods only.

Annotated Version — Read First

```yaml
# Service — redis (ClusterIP)
# Exposes Redis internally within the cluster
# Backend connects to Redis via 'redis:6379' — CoreDNS resolves 'redis' to this Service

apiVersion: v1
kind: Service
metadata:
  name: redis                   # This name is what 'REDIS_HOST: redis' in the ConfigMap resolves to
  namespace: humor-memory-game
spec:
  selector:
    app: redis                  # Routes traffic to pods with this label
  ports:
  - port: 6379                  # Port exposed by the Service
    targetPort: 6379            # Port the Redis container listens on
    protocol: TCP
  type: ClusterIP               # Internal only — not accessible outside the cluster
```

Production Copy — Run This Command

```bash
cat > 09-redis-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: humor-memory-game
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
    protocol: TCP
  type: ClusterIP
EOF
```

Verify:

```bash
cat 09-redis-service.yaml
```

---

### 10-backend-deployment.yaml

The backend Deployment runs 2 replicas of the Node.js API. Configuration is injected from the ConfigMap and all three Secrets. Prometheus scrape annotations are included so the monitoring stack can collect metrics automatically.

Annotated Version — Read First

```yaml
# Deployment — backend
# Runs 2 replicas of the Node.js API server
# Configuration injected from ConfigMap and Secrets via envFrom
# Prometheus annotations enable automatic metrics scraping

apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: humor-memory-game
  labels:
    app: backend
spec:
  replicas: 2                   # 2 replicas — load balanced across both nodes
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
      annotations:
        prometheus.io/scrape: "true"      # Prometheus will scrape this pod automatically
        prometheus.io/port: "3001"        # Port Prometheus scrapes metrics from
        prometheus.io/path: "/api/health" # Path Prometheus uses for the scrape
    spec:
      containers:
      - name: backend
        image: christie62/humor-memory-game-backend:latest
        imagePullPolicy: Always           # Always pull — ensures latest multi-platform image is used
        ports:
        - containerPort: 3001
        envFrom:
        - configMapRef:
            name: app-config             # Injects all ConfigMap keys as environment variables
        - secretRef:
            name: database-secret        # Injects DB_USER, DB_PASSWORD
        - secretRef:
            name: redis-secret           # Injects REDIS_PASSWORD
        - secretRef:
            name: app-secret             # Injects JWT_SECRET, SESSION_SECRET
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3001
          initialDelaySeconds: 30        # Allow backend time to connect to postgres and redis
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3001
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          limits:
            memory: "512Mi"             # Hard ceiling — pod is killed if exceeded
            cpu: "500m"                 # 500 millicores = 0.5 CPU cores
          requests:
            memory: "256Mi"             # Minimum memory guaranteed to the pod
            cpu: "250m"                 # Minimum CPU guaranteed to the pod
```

Production Copy — Run This Command

```bash
cat > 10-backend-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: humor-memory-game
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3001"
        prometheus.io/path: "/api/health"
    spec:
      containers:
      - name: backend
        image: christie62/humor-memory-game-backend:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 3001
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: database-secret
        - secretRef:
            name: redis-secret
        - secretRef:
            name: app-secret
        livenessProbe:
          httpGet:
            path: /api/health
            port: 3001
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/health
            port: 3001
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          limits:
            memory: "512Mi"
            cpu: "500m"
          requests:
            memory: "256Mi"
            cpu: "250m"
EOF
```

Verify:

```bash
cat 10-backend-deployment.yaml
```

---

### 11-backend-service.yaml

The backend Service exposes the API internally to the frontend pods.

Annotated Version — Read First

```yaml
# Service — backend (ClusterIP)
# Exposes the backend API internally within the cluster
# Frontend connects to the backend via 'backend:3001'

apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: humor-memory-game
spec:
  selector:
    app: backend                # Routes traffic across all running backend pod replicas
  ports:
  - port: 3001                  # Port exposed by the Service
    targetPort: 3001            # Port the backend container listens on
    protocol: TCP
  type: ClusterIP               # Internal only — frontend accesses via Nginx proxy
```

Production Copy — Run This Command

```bash
cat > 11-backend-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: humor-memory-game
spec:
  selector:
    app: backend
  ports:
  - port: 3001
    targetPort: 3001
    protocol: TCP
  type: ClusterIP
EOF
```

Verify:

```bash
cat 11-backend-service.yaml
```

---

### 12-frontend-deployment.yaml

The frontend Deployment runs 2 replicas of the Nginx static file server. The `API_BASE_URL` is injected from the ConfigMap so the frontend knows how to route API requests.

Annotated Version — Read First

```yaml
# Deployment — frontend
# Runs 2 replicas of the Nginx frontend server
# API_BASE_URL injected from ConfigMap — tells Nginx where to proxy API requests

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: humor-memory-game
  labels:
    app: frontend
spec:
  replicas: 2                   # 2 replicas — load balanced, spread across both nodes
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: christie62/humor-memory-game-frontend:latest
        imagePullPolicy: Always           # Always pull — ensures latest multi-platform image is used
        ports:
        - containerPort: 80
        env:
        - name: API_BASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: API_BASE_URL           # Injected as /api — Nginx proxies /api/* to backend
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          limits:
            memory: "256Mi"             # Frontend is lighter than backend
            cpu: "250m"
          requests:
            memory: "128Mi"
            cpu: "100m"
```

Production Copy — Run This Command

```bash
cat > 12-frontend-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: humor-memory-game
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: christie62/humor-memory-game-frontend:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        env:
        - name: API_BASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: API_BASE_URL
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          limits:
            memory: "256Mi"
            cpu: "250m"
          requests:
            memory: "128Mi"
            cpu: "100m"
EOF
```

Verify:

```bash
cat 12-frontend-deployment.yaml
```

---

### 13-frontend-service.yaml

The frontend Service is the only externally accessible service in the stack. It uses NodePort to expose the application on port 30080 across all cluster nodes.

Annotated Version — Read First

```yaml
# Service — frontend (NodePort)
# The only externally accessible service in the stack
# NodePort 30080 is opened on all nodes — browser traffic enters here
# Access the application at http://192.168.30.20:30080 from any machine on the network

apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: humor-memory-game
spec:
  selector:
    app: frontend               # Routes traffic across all running frontend pod replicas
  ports:
  - port: 80                    # Internal cluster port
    targetPort: 80              # Port the frontend container listens on
    nodePort: 30080             # External port opened on all nodes (30000–32767 range)
    protocol: TCP
  type: NodePort                # Exposes the service on a static port on every node IP
```

Production Copy — Run This Command

```bash
cat > 13-frontend-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: humor-memory-game
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
    protocol: TCP
  type: NodePort
EOF
```

Verify:

```bash
cat 13-frontend-service.yaml
```

---

### 14-ingress.yaml

> **Note:** The Ingress manifest is not applied in Phase 4. NodePort is used for external access instead. This file is included as a reference for a future phase when a domain name and ingress controller are configured.

```bash
cat > 14-ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: humor-memory-game-ingress
  namespace: humor-memory-game
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: "gameapp.local"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend
            port:
              number: 3001
EOF
```

---

## 5.5 Build Multi-Platform Docker Images

> ⚠️ **Critical:** The Raspberry Pi cluster runs ARM64. Your development machine runs x86-64 (AMD64). Images built without the `--platform` flag will not run on the Pi nodes — they crash with `exec format error`.

Both architectures must be built and pushed to Docker Hub in a single command using `docker buildx`.

### One-Time Setup on ops-box-01

SSH into the DevOps build machine:

```bash
ssh ops-box-01
```

Install QEMU and docker-buildx:

```bash
# QEMU allows x86-64 to build ARM64 images via CPU emulation
sudo pacman -S qemu-user-static

# Install docker-buildx plugin
mkdir -p ~/.docker/cli-plugins

wget https://github.com/docker/buildx/releases/download/v0.33.0/buildx-v0.33.0.linux-amd64 \
  -O ~/.docker/cli-plugins/docker-buildx

chmod +x ~/.docker/cli-plugins/docker-buildx

docker buildx version
# github.com/docker/buildx v0.33.0
```

Create the multi-architecture builder:

```bash
docker buildx create --name multiarch --driver docker-container --use

docker buildx ls
# multiarch   docker-container    running
```

### Build and Push Images

```bash
# Navigate to the application source repository
cd ~/workspace/humor-memory-game/

# Login to Docker Hub
docker login
```

Build the backend image for both AMD64 and ARM64:

Replace `<USERNAME>` with your own username

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag <USERNAME>/humor-memory-game-backend:latest \
  --push \
  -f /home/devops/workspace/humor-memory-game-devops/docker/backend/Dockerfile \
  ./backend
```

Build the frontend image for both AMD64 and ARM64:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag <USERNAME>/humor-memory-game-frontend:latest \
  --push \
  -f /home/devops/workspace/humor-memory-game-devops/docker/frontend/Dockerfile \
  ./frontend
```

### Verify Both Architectures Were Pushed

```bash
IMAGE_NAME="<USERNAME>/humor-memory-game-backend:latest"
docker buildx imagetools inspect "$IMAGE_NAME" | \
  grep "Platform:" | \
  grep -v "unknown" | \
  sed "s|Platform:[[:space:]]*|$IMAGE_NAME: |"
```

Expected output:

```
<USERNAME>/humor-memory-game-backend:latest: linux/amd64

<USERNAME>/humor-memory-game-backend:latest: linux/arm64
```

> The `unknown` entries in the manifest are attestation metadata (build provenance for security scanning). They are normal and expected — only `amd64` and `arm64` entries are actual runnable images.

---

## 5.6 Create Secrets

Secrets are created directly with `kubectl` — never stored in Git. Run from `sre-mgmnt-01`:

```bash
cd ~/humor-memory-game-kubernetes/manifests

kubectl apply -f 01-namespace.yaml

kubectl create secret generic database-secret \
  --from-literal=DB_USER=gameuser \
  --from-literal=DB_PASSWORD=gamepass123 \
  -n humor-memory-game

kubectl create secret generic redis-secret \
  --from-literal=REDIS_PASSWORD=gamepass123 \
  -n humor-memory-game

kubectl create secret generic app-secret \
  --from-literal=JWT_SECRET=dev-secret-key-change-in-production \
  --from-literal=SESSION_SECRET=dev-session-key-change-in-production \
  -n humor-memory-game
```

---

## 5.7 Deploy to the Cluster

### Prerequisite — Verify the Synology CSI Driver is Running

The application uses iSCSI persistent storage for PostgreSQL and Redis. The Synology CSI driver must be installed and all pods running before applying the application manifests — without it, the PVCs will remain in `Pending` state indefinitely and the postgres and redis pods will never start.

Verify the CSI driver is running:

```bash
kubectl get pods -n synology-csi
```

Expected output:

```
NAME                        READY   STATUS    RESTARTS   AGE
synology-csi-controller-0   4/4     Running   0          1d
synology-csi-node-jtqkq     2/2     Running   0          1d
synology-csi-node-zdzfx     2/2     Running   0          1d
```

Verify both storage classes are present:

```bash
kubectl get storageclass
```

Expected output:

```
NAME                     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
local-path (default)     rancher.io/local-path   Delete          WaitForFirstConsumer
synology-iscsi-delete    csi.san.synology.com    Delete          Immediate
synology-iscsi-storage   csi.san.synology.com    Retain          Immediate
```

> If the synology-csi pods are not running or the storage classes are missing, return to Document 4 — Section 4.8 and complete the Synology CSI driver setup before continuing.

**What happens if you skip this step:**

The PVC manifests (`04-pvc-postgres.yaml` and `05-pvc-redis.yaml`) will apply without error — Kubernetes accepts them. But the CSI driver cannot provision the LUNs on NAS2, so the PVCs stay `Pending`. PostgreSQL and Redis pods will then also stay `Pending` because their storage is never bound:

```
NAME                        READY   STATUS    RESTARTS   AGE
postgres-669d7b5879-2rssz   0/1     Pending   0          73m
redis-687776d65f-fb7qp      0/1     Pending   0          73m
```

This is the most common cause of pods stuck in `Pending` in this setup.

---

### Verify Manifest Files

Verify all manifest files are present before deploying:

```bash
cat 01-namespace.yaml
cat 03-configmap.yaml
cat 04-pvc-postgres.yaml
cat 05-pvc-redis.yaml
cat 06-postgres-deployment.yaml
cat 07-postgres-service.yaml
cat 08-redis-deployment.yaml
cat 09-redis-service.yaml
cat 10-backend-deployment.yaml
cat 11-backend-service.yaml
cat 12-frontend-deployment.yaml
cat 13-frontend-service.yaml
```

Open a second terminal and watch pods start:

```bash
kubectl get pods -n humor-memory-game --watch
```

Apply all manifests in order:

```
cd ~/humor-memory-game-kubernetes/manifests
```

```bash
kubectl apply -f 03-configmap.yaml

kubectl apply -f 04-pvc-postgres.yaml

kubectl apply -f 05-pvc-redis.yaml

kubectl apply -f 06-postgres-deployment.yaml
# wait until the postgress pod is ready and running

kubectl apply -f 07-postgres-service.yaml

kubectl apply -f 08-redis-deployment.yaml
# wait until the redis pod is ready and running

kubectl apply -f 09-redis-service.yaml

kubectl apply -f 10-backend-deployment.yaml
# wait until the backend pod is ready and running

kubectl apply -f 11-backend-service.yaml

kubectl apply -f 12-frontend-deployment.yaml
# wait until the frontend pod is ready and running

kubectl apply -f 13-frontend-service.yaml
```

```bash
kubectl get pods -n humor-memory-game
```

Expected final state:

```
NAME                        READY   STATUS    RESTARTS   AGE
backend-56d646f6bf-694hz    1/1     Running   0          2m
backend-56d646f6bf-vphwv    1/1     Running   0          2m
frontend-6876794bf4-n4tm5   1/1     Running   0          2m
frontend-6876794bf4-p2jjx   1/1     Running   0          2m
postgres-69d5b68799-shpcm   1/1     Running   0          3m
redis-7f96659d49-srln4      1/1     Running   0          3m
```

---

## 5.8 Verify the Application

Check the full application stack:

```bash
kubectl get all -n humor-memory-game
```

Expected output:

```
NAME                            READY   STATUS    RESTARTS   AGE
pod/backend-56d646f6bf-694hz    1/1     Running   0          2m22s
pod/backend-56d646f6bf-vphwv    1/1     Running   0          2m36s
pod/frontend-6876794bf4-n4tm5   1/1     Running   0          12m
pod/frontend-6876794bf4-p2jjx   1/1     Running   0          12m
pod/postgres-69d5b68799-shpcm   1/1     Running   0          3m51s
pod/redis-7f96659d49-srln4      1/1     Running   0          14m

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/backend    ClusterIP   10.43.48.180    <none>        3001/TCP       12m
service/frontend   NodePort    10.43.130.154   <none>        80:30080/TCP   12m
service/postgres   ClusterIP   10.43.155.188   <none>        5432/TCP       14m
service/redis      ClusterIP   10.43.107.41    <none>        6379/TCP       14m

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend    2/2     2            2           12m
deployment.apps/frontend   2/2     2            2           12m
deployment.apps/postgres   1/1     1            1           14m
deployment.apps/redis      1/1     1            1           14m
```

Test the application is accessible:

```bash
curl http://192.168.30.20:30080
```

Expected result
```
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>🎮 Humor Memory Game - DevOps Learning Edition 😂</title>
 
......

    </script>
  </body>
</html>
```

**Access from any machine on your network:**

- Frontend: `http://192.168.30.20:30080`
- API health: `http://192.168.30.20:30080/api/health`
- Leaderboard: `http://192.168.30.20:30080/api/leaderboard`

### End-to-End Functional Test

Open the application in your browser and verify the full stack is working:

1. Go to `http://192.168.30.20:30080`
2. Enter a username and start a game
3. Complete a game so a score is saved to PostgreSQL
4. Check the leaderboard to confirm the score was persisted
5. Refresh the page and check the leaderboard again to confirm data survived the refresh

Then verify from the backend directly:

```bash
curl http://192.168.30.20:30080/api/leaderboard
```

If the leaderboard returns your score after playing, the full stack is confirmed working — frontend → backend → PostgreSQL → Redis, all on NAS2 iSCSI storage.

---

## 5.9 Known Issues

### Known Issue — Architecture Mismatch (exec format error)

**Symptom:**

```
pod/backend-bd574855-fbzws      0/1     CrashLoopBackOff
pod/frontend-6c649658bd-g5q8b   0/1     CrashLoopBackOff
```

```bash
kubectl logs -n humor-memory-game deployment/backend
# exec /usr/local/bin/docker-entrypoint.sh: exec format error
```

**Root cause:** Images were built on the AMD64 development machine without the `--platform` flag. The ARM64 Raspberry Pi nodes cannot execute AMD64 binaries.

**Fix:** Rebuild with `docker buildx --platform linux/amd64,linux/arm64` as shown in Section 5.5.

---

### Known Issue — Redis Image Architecture Cache

**Symptom:**

Redis pod crashes with `exec format error` even after fixing backend and frontend:

```bash
kubectl logs -n humor-memory-game deployment/redis
# exec /usr/local/bin/redis-server: exec format error

kubectl describe pod -n humor-memory-game -l app=redis | grep -A 5 "Events:"
# Container image "redis:7-alpine" already present on machine
```

**Root cause:** `redis:7-alpine` (floating tag) was cached from a previous pull of the wrong architecture. Floating tags can resolve to different architectures depending on what the node has cached.

**Fix:** Pin to an explicit version tag and clear the node image cache:

```bash
# Clear image cache on both nodes
ssh cp-01 "sudo crictl rmi --prune"
ssh wrk-01 "sudo crictl rmi --prune"
```

The manifest already uses `redis:7.2-alpine`. If this was changed, revert it and redeploy:

```bash
kubectl apply -f 08-redis-deployment.yaml
kubectl get pods -n humor-memory-game -l app=redis --watch
```

> ⚠️ Do not change the Redis image tag from `redis:7.2-alpine` without testing on ARM64 nodes first.

---

### Known Issue — PostgreSQL ContainerCreating (configmap not found)

**Symptom:**

PostgreSQL pod is stuck in `ContainerCreating` and logs return:

```bash
kubectl logs -n humor-memory-game deployment/postgres
# Error from server (BadRequest): container "postgres" in pod "postgres-xxx" is waiting to start: ContainerCreating
```

Checking pod events reveals the actual cause:

```bash
kubectl describe pod -n humor-memory-game -l app=postgres | grep -A 20 "Events:"
# Warning  FailedMount  kubelet  MountVolume.SetUp failed for volume "postgres-init" : configmap "postgres-init-sql" not found
```

**Root cause:** The `06-postgres-deployment.yaml` manifest references a volume named `postgres-init` backed by a ConfigMap called `postgres-init-sql` that does not exist in the cluster. Kubernetes cannot mount a volume from a ConfigMap that is missing — the pod is held in `ContainerCreating` indefinitely.

**Distinction from Pending:** Unlike `Pending` (CSI driver not running, PVC not bound), `ContainerCreating` means the PVC bound and the iSCSI LUN attached successfully — Kubernetes is trying to start the container but cannot because a required volume is missing.

**Fix:** The `postgres-init` volumeMount and volume are not required. The backend handles database initialisation on startup. Remove both blocks from `06-postgres-deployment.yaml`:

Remove from `volumeMounts:`:
```yaml
        - name: postgres-init
          mountPath: /docker-entrypoint-initdb.d
```

Remove from `volumes:`:
```yaml
      - name: postgres-init
        configMap:
          name: postgres-init-sql
```

Then apply and restart:

```bash
kubectl apply -f 06-postgres-deployment.yaml
kubectl rollout restart deployment/postgres -n humor-memory-game
kubectl get pods -n humor-memory-game --watch
```

> **Note:** The Production Copy of `06-postgres-deployment.yaml` in Section 5.4 does not include these blocks. If you are deploying from scratch using the Production Copy command, this issue will not occur.

---

### Known Issue — PostgreSQL PGDATA Directory Conflict

**Symptom:**

PostgreSQL pod crashes on first start:

```bash
kubectl logs -n humor-memory-game deployment/postgres
# initdb: error: directory "/var/lib/postgresql/data" exists but is not empty
```

**Root cause:** iSCSI LUNs formatted as ext4 contain a `lost+found` directory at the root of the mount point. PostgreSQL refuses to initialise in a non-empty directory.

**Fix:** The `PGDATA` environment variable in `06-postgres-deployment.yaml` is set to a subdirectory of the mount point:

```yaml
- name: PGDATA
  value: /var/lib/postgresql/data/pgdata
```

This directs PostgreSQL to initialise in `/var/lib/postgresql/data/pgdata` rather than the mount root, avoiding the `lost+found` conflict. This fix is already included in the manifest.

---

### Known Issue — categories: null Validation Bug

**Symptom:** Clicking Start Game without selecting emoji categories fails with a validation error.

**Root cause:** The frontend sends `null` when no categories are selected, but the backend validation schema requires an array.

**Workaround:** Select at least one emoji category before clicking Start Game.

> For the full investigation, root cause analysis, proposed fix, and CI/CD resolution process — see `BUG_001_CATEGORIES_NULL_VALIDATION.md`.

---

## 5.10 Verification Checklist

- [ ] All 13 manifest files created in `~/humor-memory-game-kubernetes/manifests/`
- [ ] Multi-platform images built for `linux/amd64` and `linux/arm64` and pushed to Docker Hub
- [ ] Both architectures confirmed with `docker buildx imagetools inspect`
- [ ] Namespace created — `kubectl get namespace humor-memory-game`
- [ ] All three Secrets created — database-secret, redis-secret, app-secret
- [ ] All 6 pods Running — backend (×2), frontend (×2), postgres (×1), redis (×1)
- [ ] `kubectl get all -n humor-memory-game` shows all services with correct types
- [ ] Application accessible at `http://192.168.30.20:30080`
- [ ] API health check responding at `http://192.168.30.20:30080/api/health`
- [ ] End-to-end functional test complete — game played, score saved, leaderboard confirmed

---

*Next: Document — Git Workflow
