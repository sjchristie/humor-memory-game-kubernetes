# MONITORING_DASHBOARDS

## Grafana Dashboard Management

This document covers dashboard management for the Humor Memory Game monitoring stack. It is a post-installation reference — the monitoring stack must be fully installed and accessible before any steps here are followed.

The comprehensive dashboard is already loaded and configured as the home page as part of `MONITORING_SETUP.md`. This document covers importing additional dashboards, building dashboards manually, and managing existing dashboard ConfigMaps.

---

## Environment Reference

| Component | Detail |
|---|---|
| Grafana | `http://192.168.30.12:3000` |
| Grafana credentials | `admin` / `admin123` |
| Dashboard files | `monitoring/dashboards/` |
| Repository | `/home/sre/humor-memory-game-kubernetes/` |

---

## Before You Begin

### Confirm Grafana is accessible

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
```

Expected output:

```
NAME                                  READY   STATUS    RESTARTS   AGE
monitoring-grafana-56ddf4477c-tjf75   3/3     Running   0          2h
```

All three containers must show `Running` before proceeding. If port-forwards are not active, start them:

Define a "Monitoring Runner" function
```bash
run_monitoring() {
  echo "Starting Monitoring Tunnels... Press Ctrl+C to stop all."
  trap "kill 0" EXIT
  kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring --address=0.0.0.0 &
  kubectl port-forward svc/monitoring-kube-prometheus-prometheus 8081:9090 -n monitoring --address=0.0.0.0 &
  wait
}
```

Remove any existing port forwarding

```bash
killall kubectl
```

Run the following
```bash
run_monitoring
```

### Confirm dashboard files are present

```bash
ls -p monitoring/dashboards/ | grep -v /
```

Expected output:
```
advanced-custom-dashboard.json
comprehensive-dashboard.json
custom-dashboard.json
```

### Dashboard summary

| File                             | Panels | Contents                                                                                            |
| -------------------------------- | ------ | --------------------------------------------------------------------------------------------------- |
| `custom-dashboard.json`          | 2      | Pod status and HTTP request rate — minimal setup                                                    |
| `advanced-custom-dashboard.json` | 8      | Game-specific metrics — scores, errors, Redis cache, database performance                           |
| `comprehensive-dashboard.json`   | 10     | Full monitoring view — application health, HTTP requests, errors, memory, CPU, database, game stats |

---

## How the Grafana Sidecar Works

The `kube-prometheus-stack` Helm chart deploys a Grafana sidecar container (`grafana-sc-dashboard`) alongside the main Grafana container. This sidecar watches the Kubernetes API for ConfigMaps in any namespace that carry the label `grafana_dashboard=1`. When it finds one, it writes the JSON content to `/tmp/dashboards/` and Grafana's provisioning picks it up automatically.

```
ConfigMap (label: grafana_dashboard=1)
        │
        ▼
grafana-sc-dashboard (sidecar)
        │   writes JSON to /tmp/dashboards/
        ▼
Grafana provisioning
        │   reads from /tmp/dashboards/
        ▼
Dashboard appears in Grafana UI
```

Key confirmed values from implementation:

| Setting | Value |
|---|---|
| Sidecar watch label | `grafana_dashboard=1` |
| Sidecar write folder | `/tmp/dashboards/` |
| Provisioning path | `/tmp/dashboards/` |
| Datasource UID | `prometheus` |

> **Note:** The datasource UID `prometheus` is assigned by the `kube-prometheus-stack` chart. Dashboard JSON files must reference this exact UID — not `${DS_PROMETHEUS}` or any hardcoded UID from another environment. Using `${DS_PROMETHEUS}` causes all panels to show "Datasource not found" when loaded via ConfigMap provisioning, as the variable is only resolved during manual UI import.

---

## Option A: Import Additional Dashboards via the Grafana UI

**When to use this:** When you want to load the basic or advanced dashboards alongside the comprehensive dashboard already installed. No cluster changes required — the import happens entirely through the Grafana UI.

### A.1 Verify the datasource UID in the dashboard file

Before importing, confirm the dashboard JSON references the correct datasource UID:

Verify
```bash
grep '"uid"' monitoring/dashboards/custom-dashboard.json  | grep -v "Grafana" | grep -v "basic-custom-game-dashboard"
```

Expected output — all datasource UIDs showing `prometheus`:

```
        "uid": "prometheus"
        "uid": "prometheus"
        "uid": "prometheus"
        ...
```

If the output shows `${DS_PROMETHEUS}` or another value, the file needs to be corrected before importing. During a manual UI import Grafana resolves `${DS_PROMETHEUS}` automatically — but verifying first avoids confusion if the file is later used in a ConfigMap.

If any line shows `${DS_PROMETHEUS}` or a hardcoded UID from another environment, replace all datasource UIDs with `prometheus`:

```bash
sed -i 's/\${DS_PROMETHEUS}/prometheus/g' monitoring/dashboards/custom-dashboard.json
```

Verify again before continuing.


### A.2 Import the basic dashboard

1. In Grafana, click **Dashboards** in the right sidebar
2. Click **New** → **Import dashboard**
3. Click **Upload dashboard JSON file**
4. Select `monitoring/dashboards/custom-dashboard.json` from your local machine
5. Enter Name 'My Custom Game Dashboard'
6. Click **Import**


> **Note:** If dashboard panels show no data after importchanges the Data source to 'Grafana'

---

## Option B: Load a Dashboard via ConfigMap

**When to use this:** When you want a dashboard to load automatically via the Grafana sidecar — the same mechanism used for the comprehensive dashboard in `MONITORING_SETUP.md`. Use this when adding a new dashboard that should survive pod restarts and redeployments.

### B.1 Verify the datasource UID

```bash
grep '"uid"' monitoring/dashboards/advanced-custom-dashboard.json  | grep -v "Grafana" | grep -v "advanced-custom-game-dashboard"
```

Expected output — all UIDs must show `prometheus`:

```
        "uid": "prometheus"
        "uid": "prometheus"
```

If any line shows `${DS_PROMETHEUS}` or another value, correct the file before continuing:

```bash
sed -i -E 's/"uid":\s*"(\$\{DS_PROMETHEUS\}|PBFA97CFB590B2093)"/"uid": "prometheus"/g' monitoring/dashboards/advanced-custom-dashboard.json
```

Verify again before proceeding.

### B.2 Create the ConfigMap

```bash
kubectl create configmap advanced-game-dashboard \
  --from-file=advanced-custom-dashboard.json=monitoring/dashboards/advanced-custom-dashboard.json \
  --namespace monitoring
```

Expected output:
```
configmap/advanced-game-dashboard created
```

### B.3 Apply the sidecar label

```bash
kubectl label configmap advanced-game-dashboard -n monitoring grafana_dashboard=1
```

Expected output:
```
configmap/advanced-game-dashboard labeled
```

### B.4 Verify the dashboard was loaded

Confirm the sidecar wrote the file to `/tmp/dashboards/`:

```bash
kubectl exec -n monitoring deploy/monitoring-grafana -c grafana-sc-dashboard -- \
  ls -la /tmp/dashboards/ | grep advanced | awk '{print $NF}'
```

Expected output:
```
advanced-custom-dashboard.json
```

Then open Grafana at `http://192.168.30.12:3000`, click **Dashboards**, and confirm the dashboard appears in the list.

There will be issue with No data, ignore this as this just demonstration the process

---

## Option C: Build a Dashboard Manually

**When to use this:** When you want to understand how Grafana dashboards are constructed by building panels from scratch using PromQL queries against real metrics.

The backend exposes default Node.js process metrics via `prom-client`. The queries below use the metrics confirmed available during implementation.

### C.1 Create a new dashboard

1. In Grafana, on the right
2. Click **New** → **New Dashboard**
3. Click **Add visualisation**
4. Select **Prometheus** as the data source

### C.2 Panel 1 — Backend CPU Usage

In the query editor:

```
rate(process_cpu_seconds_total{namespace="humor-memory-game"}[5m]) * 100
```

Panel settings:
- Title: `Backend CPU Usage`
- Visualization: `Time series`
- Unit: `Percent (0-100)`

Click **Apply**.

### C.3 Panel 2 — Backend Memory Usage

In the query editor:

```
process_resident_memory_bytes{namespace="humor-memory-game"}
```

Panel settings:
- Title: `Backend Memory Usage`
- Visualization: `Time series`
- Unit: `bytes`

Click **Apply**.

### C.4 Panel 3 — Node.js Event Loop Lag

In the query editor:

```
nodejs_eventloop_lag_seconds{namespace="humor-memory-game"}
```

Panel settings:
- Title: `Event Loop Lag`
- Visualization: `Time series`
- Unit: `seconds`

Click **Apply**.

### C.5 Panel 4 — Pod Status

In the query editor:

```
kube_pod_status_phase{namespace="humor-memory-game"}
```

Panel settings:
- Title: `Pod Status`
- Visualization: `Stat`

Click **Apply**.

### C.6 Save the dashboard

Click the **Save dashboard** icon in the top right. Give the dashboard a meaningful name and click **Save**.

> **Note:** Dashboards saved through the Grafana UI are stored in Grafana's internal database. They do not persist if the Grafana pod is deleted or the monitoring namespace is removed. To make a dashboard persistent, export it as JSON from **Dashboard Settings → JSON Model**, save to `monitoring/dashboards/`, and load it via ConfigMap following Option B.

---

## Managing Existing Dashboard ConfigMaps

### List all dashboard ConfigMaps

```
kubectl get configmaps -n monitoring -l grafana_dashboard=1
```

Expected output:

```
NAME                           DATA   AGE
comprehensive-game-dashboard   1      2h
```

### Update a dashboard ConfigMap

If you have modified a dashboard JSON file and want to reload it:

```
kubectl delete configmap comprehensive-game-dashboard -n monitoring
```

```
kubectl create configmap comprehensive-game-dashboard \
  --from-file=comprehensive-dashboard.json=monitoring/dashboards/comprehensive-dashboard.json \
  --namespace monitoring
```

```
kubectl label configmap comprehensive-game-dashboard -n monitoring grafana_dashboard=1
```

The sidecar detects the new ConfigMap and reloads the dashboard within 60 seconds.

### Remove a dashboard ConfigMap

```
kubectl delete configmap comprehensive-game-dashboard -n monitoring
```

The sidecar removes the dashboard from Grafana automatically within 60 seconds.

---

## Next Steps

| Document | Contents |
|---|---|
| `MONITORING_METRICS.md` | PromQL queries, metric exploration, verifying scrape targets |
| `MONITORING_TROUBLESHOOTING.md` | Ingress routing issues, SSH tunnel errors, RBAC 403, datasource issues |
| `MONITORING_CUSTOM_DASHBOARDS.md` | Building and customising dashboards from scratch |
