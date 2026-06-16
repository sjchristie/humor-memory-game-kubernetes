# Humor Memory Game — Kubernetes Infrastructure

> **Phase 4 of a DevOps project** — a full-stack web application containerised with Docker and deployed to a production-grade self-hosted Kubernetes cluster running on Raspberry Pi hardware.

---

## What This Project Demonstrates

This repository contains the complete Kubernetes infrastructure for a full-stack web application, deployed to a self-hosted 2-node K3s cluster on Raspberry Pi hardware. It represents the kind of real-world DevOps work done daily at engineering teams — not a cloud-managed service doing the heavy lifting, but hands-on infrastructure built from scratch.

### The Three Phases

| Phase                | Repository                                                           | What Was Built                                                              |
| -------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| 1 — Development      | [humor-memory-game](https://github.com/sjchristie/humor-memory-game) | Full-stack app: Node.js API, React frontend, PostgreSQL, Redis              |
| 2 — Containerisation | [humor-memory-game](https://github.com/sjchristie/humor-memory-game) | Docker, multi-stage builds<br>Docker Compose, multi-container orchestration |
| 3 — Kubernetes       | **This repository**                                                  | K3s cluster, GitOps, monitoring, persistent storage                         |

---

## Technical Highlights

### Self-Hosted Kubernetes Cluster on Raspberry Pi

A 2-node K3s cluster running on physical Raspberry Pi hardware — not a VM, not a cloud provider. Control plane and worker node separation, persistent iSCSI storage via Synology NAS, service discovery, health checks, and automatic pod restarts.

- **cp-01** — Raspberry Pi 5 (8GB) — Control Plane
- **wrk-01** — Raspberry Pi 4 (4GB) — Worker Node

### Cross-Platform Docker Builds (ARM64 + AMD64)

One of the most technically interesting problems solved in this project.

The development machine runs **x86-64 (AMD64)**. The Raspberry Pi cluster runs **ARM64**. Docker images built on one architecture will not run on the other — they crash with `exec format error`.

**Solution:** `docker buildx` with QEMU emulation to build multi-platform images in a single pipeline step, published to Docker Hub as a single manifest tag that automatically serves the correct architecture to each platform.

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag christie62/humor-memory-game-backend:latest \
  --push \
  -f backend/Dockerfile ./backend
```

This is the same pattern used in professional CI/CD pipelines for edge and IoT deployments.

### GitOps with Flux

- Git is the single source of truth for cluster state
- Flux watches this repository and reconciles any drift between Git and the live cluster
- Manual reconciliation mode during Phase 4 (safe pattern — no auto-prune)
- Kustomize overlays for environment separation (dev/staging/prod ready)
- Auto-sync enabled in Phase 5 CI/CD pipeline

### Infrastructure as Code

All Kubernetes resources defined declaratively across 13 manifest files:

- Namespace isolation
- ConfigMaps for non-sensitive configuration
- Secrets for credentials (never stored in Git)
- PersistentVolumeClaims backed by Synology iSCSI NAS storage
- Deployments with health checks and resource limits
- Services for internal networking
- Ingress reference for future phase

### Observability Stack

- Prometheus for metrics collection — 5 scrape jobs covering nodes, containers, and application pods
- kube-state-metrics for Kubernetes object state (pod status, deployment replicas, PVC usage)
- Grafana for dashboards — Pi K3s Cluster Overview and Humor Memory Game dashboards
- RBAC-scoped monitoring — cluster metrics without cluster-admin access

### Persistent Storage via Synology CSI Driver

Application data is stored on a Synology NAS via iSCSI, completely separate from the Pi nodes. When a pod moves between nodes — due to failure, maintenance, or rescheduling — the CSI driver detaches the iSCSI LUN from one node and reattaches it to the other. Pod mobility without data loss.

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

Cluster:
├── cp-01 (192.168.30.20) — Raspberry Pi 5 — Control Plane
└── wrk-01 (192.168.30.21) — Raspberry Pi 4 — Worker Node

Management:
└── sre-mgmnt-01 — Arch Linux VM — kubectl / Flux / Helm
```

---

## Key Engineering Decisions

| Decision | Why |
|----------|-----|
| K3s over full Kubernetes | 75% less RAM on constrained hardware — 400MB vs 1.5GB — while remaining CNCF-certified and 100% `kubectl` compatible |
| Containerd over Docker runtime | Lighter weight on nodes (150MB vs 500MB) — the direction the industry has moved |
| Synology CSI driver over local-path | Pod mobility — data lives on NAS2, not tied to a specific node. Survives node failure and rescheduling |
| Flux over ArgoCD | 160MB vs 500MB memory footprint — critical on 4GB Raspberry Pi |
| Multi-platform builds | Required for ARM64 cluster — same pattern used in production edge and IoT deployments |
| Manual sync in Phase 4 | Safety-first learning pattern — auto-sync enabled in Phase 5 CI/CD pipeline |
| systemd-networkd over NetworkManager | Eliminates ghost DHCP IP addresses that interfere with Kubernetes pod networking |
| VLAN isolation (VLAN 30) | Cluster nodes on a dedicated Home Lab network segment — separated from default home network traffic |

---

## Problems Solved

**Architecture mismatch** — Images built on AMD64 crashed on ARM64 with `exec format error`. Solved by implementing multi-platform builds with QEMU emulation using `docker buildx`.

**Redis image architecture cache** — Floating image tag pulled wrong architecture from node cache. Solved by pinning to an explicit version tag and clearing the image cache on both Pi nodes with `crictl rmi --prune`.

**PostgreSQL PGDATA conflict** — iSCSI ext4 LUNs contain a `lost+found` directory at the mount root. PostgreSQL refuses to initialise in a non-empty directory. Solved by setting `PGDATA` to a subdirectory of the mount point.

**PostgreSQL search_path mismatch** — The `gameuser` role could not find database tables created in the `public` schema. Under Kubernetes's stricter role isolation, PostgreSQL's default `search_path` did not fall through to `public` reliably. Solved by running `ALTER ROLE gameuser SET search_path TO public` and granting full privileges on all tables and sequences.

**Backend probe death spiral** — Liveness and readiness probes hitting `/api/health` at high frequency exhausted the database connection pool, causing the health check itself to fail and triggering a continuous restart cycle. Solved by reducing probe frequency — liveness from 10s to 30s, readiness from 5s to 15s — with appropriate initial delay buffers.

**Prometheus RBAC 403** — Metrics collection returned HTTP 403 on node and cAdvisor endpoints. Solved by adding `nonResourceURLs` permissions for `/metrics` and `/metrics/cadvisor` to the Prometheus ClusterRole.

**iSCSI permission denied** — Prometheus (UID 65534) and Grafana (UID 472) could not write to root-owned iSCSI ext4 mounts. Solved by adding `initContainers` to set correct ownership before the main container starts.

**NetworkManager conflicts** — Secondary DHCP addresses appeared on nodes alongside static IPs, breaking pod networking. Solved by purging NetworkManager and switching to systemd-networkd with explicit static IP configuration.

---

## Repository Structure

```
humor-memory-game-kubernetes/
├── manifests/                     13 Kubernetes resource definitions
├── base/                          Kustomize base layer
├── overlays/dev/                  Development environment overlay
├── flux/                          Flux GitOps configuration
├── monitoring/                    Prometheus and Grafana manifests
├── docs/
│   ├── SRE_SETUP_GUIDE.md
│   ├── INTRODUCTION_TO_KUBERNETES.md
│   ├── PREPARING_THE_RASPBERRY_PIS.md
│   ├── INSTALLING_K3S.md
│   ├── DEPLOYING_THE_APPLICATION.md
│   ├── MONITORING_PROMETHEUS.md
│   ├── MONITORING_GRAFANA.md
│   ├── GITOPS_WITH_FLUX.md
│   ├── TROUBLESHOOTING_AND_OPERATIONS.md
│   ├── BUG_001_CATEGORIES_NULL_VALIDATION.md
│   ├── BUG_002_BACKEND_PROBE_DEATH_SPIRAL.md
│   └── BUG_003_POSTGRES_SEARCH_PATH.md
└── README.md                      This file
```

---

## Skills Demonstrated

`Kubernetes` `K3s` `GitOps` `Flux` `Kustomize` `Helm` `Docker` `Containerd` `ARM64` `Multi-platform builds` `Prometheus` `Grafana` `kube-state-metrics` `RBAC` `Persistent Volumes` `iSCSI` `Synology CSI` `NodePort` `Raspberry Pi` `Infrastructure as Code` `Linux` `Networking` `Bash` `VLAN` `systemd-networkd`

---

## Related Repositories

- **Application source:**  [Humor Memory Game - Kubernetes](https://github.com/sjchristie/humor-memory-game-kubernetes)
- **Docker repository:**   [Humor Memory Game - Frontend](https://hub.docker.com/repository/docker/christie62/humor-memory-game-frontend)
					[Humor Memory Game - Backend](https://hub.docker.com/repository/docker/christie62/humor-memory-game-backend)







