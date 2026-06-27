# Humor Memory Game — Kubernetes Infrastructure & Observability

> **Phase 3 and Phase 4 of a DevOps project** — a full-stack web application deployed to a production-grade self-hosted Kubernetes cluster on Raspberry Pi hardware, with a complete observability stack using the Prometheus Operator pattern.

---

## What This Repository Covers

This repository contains two phases of the Homelab to Production portfolio series:

**Phase 3 — Kubernetes:** The complete Kubernetes infrastructure for the full-stack application — deployed to a self-hosted 2-node K3s cluster on Raspberry Pi hardware. Hand-written manifests, persistent iSCSI storage via Synology NAS, and GitOps management via Flux CD.

**Phase 4 — Observability:** A production-grade monitoring stack deployed using the `kube-prometheus-stack` Helm chart — Prometheus Operator pattern, ServiceMonitor-based scraping, Grafana dashboards provisioned via ConfigMap, and eleven real implementation issues documented and resolved.

### The Five Phases

| Phase | Repository | What Was Built |
| --- | --- | --- |
| 1 — Development | [humor-memory-game](https://github.com/sjchristie/humor-memory-game) | Full-stack app: Node.js API, React frontend, PostgreSQL, Redis |
| 2 — Containerisation | [humor-memory-game-devops](https://github.com/sjchristie/humor-memory-game-devops) | Docker, multi-stage builds, Docker Compose, multi-container orchestration |
| 3 — Kubernetes | **This repository** | K3s cluster, persistent iSCSI storage, GitOps via Flux |
| 4 — Observability | **This repository** (`monitoring/`) | Prometheus Operator, Grafana, ServiceMonitor, kube-prometheus-stack |
| 5 — GitOps | [humor-memory-game-gitops](https://github.com/sjchristie/humor-memory-game-gitops) | ArgoCD, Kustomize overlays, GitHub Actions CI/CD pipeline |

---

## Technical Highlights

### Self-Hosted Kubernetes Cluster on Raspberry Pi

A 2-node K3s cluster running on physical Raspberry Pi hardware — not a VM, not a cloud provider. Control plane and worker node separation, persistent iSCSI storage via Synology NAS, service discovery, health checks, and automatic pod restarts.

- **cp-01** — Raspberry Pi 5 (8GB) — Control Plane
- **wrk-01** — Raspberry Pi 4 (4GB) — Worker Node
- **sre-mgmt-01** — Arch Linux VM — Management machine running `kubectl`, Helm, and Flux CLI

### Prometheus Operator Pattern

The observability stack uses the `kube-prometheus-stack` Helm chart — the industry-standard approach for Kubernetes monitoring. One Helm install deploys the full stack. Scrape targets are declared as `ServiceMonitor` resources rather than hardcoded Prometheus configuration — the Operator handles reconciliation automatically.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: humor-game-monitor
  namespace: monitoring
  labels:
    release: monitoring
spec:
  namespaceSelector:
    matchNames:
      - humor-memory-game
  selector:
    matchLabels:
      app: backend
  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
```

### Grafana Sidecar Dashboard Provisioning

Dashboards are version-controlled as JSON files and loaded into Grafana automatically via ConfigMap provisioning. The Grafana sidecar container watches for ConfigMaps labelled `grafana_dashboard=1` and loads them without manual import steps. Dashboards survive pod restarts and redeployments.

### Cross-Platform Docker Builds (ARM64 + AMD64)

The development machine runs x86-64. The Raspberry Pi cluster runs ARM64. `docker buildx` with QEMU emulation produces multi-platform images published to Docker Hub as a single manifest tag that automatically serves the correct architecture.

### Persistent Storage via Synology CSI Driver

Application data is stored on a Synology NAS via iSCSI. When a pod moves between nodes, the CSI driver detaches the iSCSI LUN from one node and reattaches it to the other. Pod mobility without data loss.

---

## Architecture

```
Your Browser
     │
     ▼
NodePort :30080 (cp-01 — 192.168.30.20)
     │
     ▼
Frontend Service (2 replicas — Nginx)
     │
     └── /api ──────► Backend Service (2 replicas — Node.js)
                            │
                    ┌───────┴────────┐
                    ▼                ▼
              PostgreSQL          Redis
           (iSCSI — NAS2)    (iSCSI — NAS2)

Observability:
├── Prometheus  (port-forward: 192.168.30.12:8081)
└── Grafana     (port-forward: 192.168.30.12:3000)

Cluster:
├── cp-01 (192.168.30.20) — Raspberry Pi 5 — Control Plane
└── wrk-01 (192.168.30.21) — Raspberry Pi 4 — Worker Node

Management:
└── sre-mgmt-01 (192.168.30.12) — Arch Linux VM — kubectl / Helm / Flux
```

---

## Key Engineering Decisions

| Decision | Why |
|---|---|
| K3s over full Kubernetes | 75% less RAM on constrained hardware — 400MB vs 1.5GB — while remaining CNCF-certified |
| Prometheus Operator over raw manifests | ServiceMonitor pattern — scrape targets declared as Kubernetes resources, no ConfigMap editing |
| Grafana sidecar provisioning | Dashboards version-controlled and redeployment-safe — no manual import |
| Containerd over Docker runtime | Lighter weight on nodes — the direction the industry has moved |
| Synology CSI driver over local-path | Pod mobility — data lives on NAS, not tied to a specific node |
| Flux over ArgoCD (Phase 3) | 160MB vs 500MB memory footprint — critical on 4GB Raspberry Pi |
| Port-forwarding over Ingress for monitoring access | No DNS changes required — `--address=0.0.0.0` makes services accessible from any network machine |
| VLAN isolation (VLAN 30) | Cluster nodes on dedicated home lab network segment |

---

## Problems Solved

### Phase 3 — Kubernetes

**PostgreSQL PGDATA conflict** — iSCSI ext4 LUNs contain a `lost+found` directory at the mount root. PostgreSQL refuses to initialise in a non-empty directory. Solved by setting `PGDATA` to a subdirectory of the mount point.

**PostgreSQL search_path mismatch** — The `gameuser` role could not find tables in the `public` schema under Kubernetes's stricter role isolation. Solved by `ALTER ROLE gameuser SET search_path TO public`.

**Backend probe death spiral** — Liveness and readiness probes exhausted the database connection pool, causing health checks to fail and triggering a continuous restart cycle. Solved by reducing probe frequency with longer initial delay buffers.

**Prometheus RBAC 403** — Metrics collection returned HTTP 403 on node and cAdvisor endpoints. Solved by adding `nonResourceURLs` permissions for `/metrics` to the Prometheus ClusterRole.

### Phase 4 — Observability

**Dashboard datasource UID** — Dashboard JSON contained a hardcoded UID from another environment. The chart assigns the UID `prometheus` — any other value causes all panels to show "Datasource not found" when loaded via ConfigMap provisioning.

**Home dashboard path** — `default_home_dashboard_path` was set to `/var/lib/grafana/dashboards/default/`. The sidecar writes to `/tmp/dashboards/` — confirmed via `kubectl describe`. Correcting the path fixed the home page immediately.

**ServiceMonitor 0/0 up** — Three compounding issues: missing `namespaceSelector` (ServiceMonitor looked in the wrong namespace), missing `app: backend` label on the backend Service (selector found nothing), unnamed service port (ServiceMonitor requires named port references).

**`kubectl edit` failing** — `vi` not installed on the management VM. All live resource changes use `kubectl patch` or file-based `kubectl apply`.

---

## Repository Structure

```
humor-memory-game-kubernetes/
├── manifests/                          13 Kubernetes resource definitions
│   ├── 01-namespace.yaml
│   ├── 03-configmap.yaml
│   ├── 04-pvc-postgres.yaml
│   ├── 05-pvc-redis.yaml
│   ├── 06-postgres-deployment.yaml
│   ├── 07-postgres-service.yaml
│   ├── 08-redis-deployment.yaml
│   ├── 09-redis-service.yaml
│   ├── 10-backend-deployment.yaml
│   ├── 11-backend-service.yaml         ← Updated Phase 4: added label + named port
│   ├── 12-frontend-deployment.yaml
│   ├── 13-frontend-service.yaml
│   └── 14-ingress.yaml
├── monitoring/                         Phase 4 — Observability stack
│   ├── README.md                       Monitoring directory entry point
│   ├── monitoring-values.yaml          Helm values for kube-prometheus-stack
│   ├── prometheus-rbac.yaml            ClusterRole, ClusterRoleBinding, ServiceAccount
│   ├── humor-service-monitor.yaml      ServiceMonitor for backend scraping
│   └── dashboards/
│       ├── comprehensive-dashboard.json   10-panel full monitoring view
│       ├── advanced-custom-dashboard.json 8-panel game-specific metrics
│       └── custom-dashboard.json          2-panel minimal dashboard
├── docs/
│   ├── SRE_SETUP_GUIDE.md
│   ├── INTRODUCTION_TO_KUBERNETES.md
│   ├── PREPARING_THE_RASPBERRY_PIS.md
│   ├── INSTALLING_K3S.md
│   ├── DEPLOYING_THE_APPLICATION.md
│   ├── MONITORING_PROMETHEUS.md        Phase 3 — raw manifest Prometheus deployment
│   ├── MONITORING_GRAFANA.md           Phase 3 — raw manifest Grafana deployment
│   ├── GITOPS_WITH_FLUX.md
│   ├── TROUBLESHOOTING_AND_OPERATIONS.md
│   ├── BUG_001_CATEGORIES_NULL_VALIDATION.md
│   ├── BUG_002_BACKEND_PROBE_DEATH_SPIRAL.md
│   ├── BUG_003_POSTGRES_SEARCH_PATH.md
│   ├── MONITORING_SETUP.md             Phase 4 — Helm-based observability installation
│   ├── MONITORING_DASHBOARDS.md        Phase 4 — Dashboard management
│   ├── MONITORING_METRICS.md           Phase 4 — PromQL reference and metric exploration
│   ├── MONITORING_TROUBLESHOOTING.md   Phase 4 — 11 confirmed issues with fixes
│   └── MONITORING_CUSTOM_DASHBOARDS.md Phase 4 — Building custom dashboards
└── README.md                           This file
```

---

## Skills Demonstrated

`Kubernetes` `K3s` `Helm` `Prometheus Operator` `kube-prometheus-stack` `ServiceMonitor` `Grafana` `PromQL` `GitOps` `Flux` `Kustomize` `Docker` `Containerd` `ARM64` `Multi-platform builds` `Persistent Volumes` `iSCSI` `Synology CSI` `RBAC` `ConfigMap provisioning` `Raspberry Pi` `Infrastructure as Code` `Linux` `Networking` `Bash` `VLAN` `systemd-networkd`

---

## Related Repositories

- **Application source:** [humor-memory-game](https://github.com/sjchristie/humor-memory-game)
- **Docker repository:** [humor-memory-game-devops](https://github.com/sjchristie/humor-memory-game-devops)
- **GitOps repository:** [humor-memory-game-gitops](https://github.com/sjchristie/humor-memory-game-gitops)
- **Docker Hub — Frontend:** [christie62/humor-memory-game-frontend](https://hub.docker.com/repository/docker/christie62/humor-memory-game-frontend)
- **Docker Hub — Backend:** [christie62/humor-memory-game-backend](https://hub.docker.com/repository/docker/christie62/humor-memory-game-backend)
