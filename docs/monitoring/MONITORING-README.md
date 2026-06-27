# Humor Memory Game — Observability Stack

> **Phase 4 of a DevOps project** — a full-stack web application monitored with a production-grade observability stack using the Prometheus Operator pattern, deployed to a self-hosted Kubernetes cluster on Raspberry Pi hardware.

---

## What This Directory Contains

This directory contains all configuration, scripts, and dashboard definitions for the Phase 4 Observability implementation. The monitoring stack is deployed using the `kube-prometheus-stack` Helm chart — the Prometheus Operator pattern — which manages Prometheus, Grafana, Alertmanager, kube-state-metrics, and node-exporter as a single managed installation.

---

## Directory Structure

```
monitoring/
├── README.md                          This file
├── monitoring-values.yaml             Helm values — Grafana password, Ingress, home dashboard path
├── prometheus-rbac.yaml               ClusterRole, ClusterRoleBinding, ServiceAccount for Prometheus
├── humor-service-monitor.yaml         ServiceMonitor — scrapes the backend /metrics endpoint
└── dashboards/
    ├── comprehensive-dashboard.json   10-panel full monitoring view (recommended — loaded by default)
    ├── advanced-custom-dashboard.json 8-panel game-specific metrics dashboard
    └── custom-dashboard.json          2-panel minimal dashboard — pod status and HTTP request rate
```

---

## Environment Reference

| Component | Detail |
|---|---|
| Management VM | `sre-mgmt-01` — `192.168.30.12` |
| Control plane | `cp-01` — `192.168.30.20` — Raspberry Pi 5 |
| Worker node | `wrk-01` — `192.168.30.21` — Raspberry Pi 4 |
| Application namespace | `humor-memory-game` |
| Monitoring namespace | `monitoring` |
| Grafana | `http://192.168.30.12:3000` |
| Prometheus | `http://192.168.30.12:8081` |
| Grafana credentials | `admin` / `admin123` |
| Prometheus datasource UID | `prometheus` |
| Dashboard sidecar folder | `/tmp/dashboards/` |

---

## Quick Start

Follow the documents in this order. Do not skip steps — each document assumes the previous is complete.

**1. Install the monitoring stack:**

```
cd /home/sre/humor-memory-game-kubernetes
```

Follow `docs/MONITORING_SETUP.md` from start to finish.

**2. Start port-forwards for browser access:**

```
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring --address=0.0.0.0 > /dev/null 2>&1 &

kubectl port-forward svc/monitoring-kube-prometheus-prometheus 8081:9090 -n monitoring --address=0.0.0.0 > /dev/null 2>&1 &
```

**3. Open in browser:**

- Grafana: `http://192.168.30.12:3000`
- Prometheus: `http://192.168.30.12:8081`

---

## Document Index

All monitoring documentation lives in `docs/`. Follow in this order:

| Document | Purpose | When to open |
|---|---|---|
| [`MONITORING_SETUP.md`](../docs/MONITORING_SETUP.md) | Complete installation guide — single linear flow from nothing to a working monitoring stack with dashboards | Start here |
| [`MONITORING_DASHBOARDS.md`](../docs/MONITORING_DASHBOARDS.md) | Dashboard management — importing additional dashboards, loading via ConfigMap, building manually | After setup is complete |
| [`MONITORING_METRICS.md`](../docs/MONITORING_METRICS.md) | Prometheus metric exploration, PromQL query reference, scrape target verification | When exploring metrics |
| [`MONITORING_TROUBLESHOOTING.md`](../docs/MONITORING_TROUBLESHOOTING.md) | 11 confirmed operational issues with root causes and fixes | When something breaks |
| [`MONITORING_CUSTOM_DASHBOARDS.md`](../docs/MONITORING_CUSTOM_DASHBOARDS.md) | Building custom dashboards from scratch — panel types, variables, annotations, export and persistence | When customising dashboards |

---

## Key Configuration Files

### `monitoring-values.yaml`

Helm values file used to install and upgrade the `kube-prometheus-stack`. Key settings:

| Setting | Value | Purpose |
|---|---|---|
| `grafana.adminPassword` | `admin123` | Fixed Grafana admin password — prevents chart generating a random password |
| `grafana.ingress.hosts` | `grafana.gameapp.local` | Grafana Ingress hostname |
| `default_home_dashboard_path` | `/tmp/dashboards/comprehensive-dashboard.json` | Sets the Comprehensive Game Monitoring Dashboard as the Grafana home page |

> **Note:** The `default_home_dashboard_path` must reference `/tmp/dashboards/` — the folder the Grafana sidecar writes to. Using `/var/lib/grafana/dashboards/default/` causes the home dashboard setting to silently fail.

### `humor-service-monitor.yaml`

ServiceMonitor that tells the Prometheus Operator to scrape the backend application. Key requirements confirmed during implementation:

- `namespaceSelector` targeting `humor-memory-game` — required because the ServiceMonitor is in `monitoring` and the backend is in `humor-memory-game`
- `port: metrics` — references the named port on the backend Service, not the port number
- `release: monitoring` label — required for the Prometheus Operator to include this ServiceMonitor

### `prometheus-rbac.yaml`

RBAC configuration for Prometheus cluster-wide read access. Includes `nonResourceURLs: ["/metrics"]` required for K3s node and cAdvisor metric endpoints.

---

## Dashboard Files

All dashboard JSON files must reference the Prometheus datasource UID `prometheus`. Files containing `${DS_PROMETHEUS}` or a hardcoded UID from another environment will cause all panels to show "Datasource not found" when loaded via ConfigMap provisioning.

Verify before use:

```
grep '"uid"' monitoring/dashboards/comprehensive-dashboard.json | grep -v "Grafana"
```

Expected output — all values must show `prometheus`:

```
        "uid": "prometheus"
```

---

## What the Monitoring Stack Covers

### Application monitoring

- Backend process CPU and memory usage via `prom-client` default metrics
- Node.js event loop lag and heap utilisation
- Pod health and restart counts via kube-state-metrics
- HTTP request activity from Kubernetes health probes

### Cluster monitoring

- Node CPU and memory usage via node-exporter on both Raspberry Pi nodes
- Container resource usage via cAdvisor
- Kubernetes object state via kube-state-metrics — pod phases, deployment replicas, PVC usage
- Control plane component health — API server, scheduler, controller-manager, etcd, CoreDNS

### What is not yet monitored

Custom application metrics (`http_requests_total`, `game_scores_total`, `database_connections_current`) are not instrumented. The backend uses default `prom-client` metrics only. Adding custom metrics requires modifying the backend source code in `sjchristie/humor-memory-game`. See the Future Enhancements section in `docs/MONITORING_METRICS.md`.

---

## Implementation Issues Resolved

The following issues were encountered and resolved during implementation. Each is documented in `docs/MONITORING_TROUBLESHOOTING.md`:

| Issue | Summary |
|---|---|
| ISSUE-MON-001 | Dashboard datasource UID hardcoded or using `${DS_PROMETHEUS}` — not resolved via ConfigMap provisioning |
| ISSUE-MON-002 | Home dashboard showing Welcome screen — wrong `default_home_dashboard_path` in values file |
| ISSUE-MON-003 | Sidecar logs show `Loading incluster config...` only — normal behaviour, not an error |
| ISSUE-MON-004 | ServiceMonitor 0/0 up — missing `namespaceSelector`, missing service label, unnamed port |
| ISSUE-MON-005 | Prometheus RBAC 403 on node metrics — missing `nonResourceURLs` in ClusterRole |
| ISSUE-MON-006 | Dashboard panels show No data — custom application metrics not instrumented |
| ISSUE-MON-007 | Ingress 404 on IP-based access — host-based routing rejects requests without hostname |
| ISSUE-MON-008 | SSH tunnel Connection refused — nothing listening on management VM port 8080 |
| ISSUE-MON-009 | Port-forward stops after `helm upgrade` — pod restart breaks port-forward binding |
| ISSUE-MON-010 | `kubectl edit` fails — `vi` not installed, use `kubectl patch` |
| ISSUE-MON-011 | `wget` not available in backend container — use `curl` |

---

## Related Repositories

| Repository | Contents |
|---|---|
| [humor-memory-game](https://github.com/sjchristie/humor-memory-game) | Application source — Node.js backend, React frontend |
| [humor-memory-game-devops](https://github.com/sjchristie/humor-memory-game-devops) | Phase 2 — Docker and Docker Compose infrastructure |
| [humor-memory-game-kubernetes](https://github.com/sjchristie/humor-memory-game-kubernetes) | Phase 3 and 4 — Kubernetes manifests and observability stack |
| [humor-memory-game-gitops](https://github.com/sjchristie/humor-memory-game-gitops) | Phase 5 — ArgoCD, Kustomize overlays, GitHub Actions |
