# MONITORING_METRICS

## Metrics and PromQL Reference

This document covers Prometheus metric exploration, PromQL queries, and scrape target verification for the Humor Memory Game monitoring stack. It is a post-installation reference — the monitoring stack must be fully installed and accessible before any steps here are followed.

Prometheus is accessible at `http://192.168.30.12:8081` when the port-forward is active.

---

## Environment Reference

| Component | Detail |
|---|---|
| Prometheus UI | `http://192.168.30.12:8081` |
| Prometheus API | `http://192.168.30.12:8081/api/v1/` |
| Scrape interval | 15 seconds |
| Application namespace | `humor-memory-game` |
| Monitoring namespace | `monitoring` |

---

## Before You Begin

Confirm the Prometheus port-forward is active:

```
jobs | grep prometheus
```

Expected output:

```
[2]  Running  kubectl port-forward svc/monitoring-kube-prometheus-prometheus 8081:9090 -n monitoring --address=0.0.0.0
```

If it is not running, start it:

```
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 8081:9090 -n monitoring --address=0.0.0.0 > /dev/null 2>&1 &
```

Open the Prometheus UI at `http://192.168.30.12:8081` and confirm it loads.

---

## Verify Scrape Targets

### Check all targets via the UI

Navigate to **Status → Target health** in the Prometheus UI.

All targets should show **UP**. The following targets are expected in a healthy installation:

| ServiceMonitor | Expected targets | State |
|---|---|---|
| `humor-game-monitor` | 1 (backend pod) | UP |
| `monitoring-grafana` | 1 | UP |
| `monitoring-kube-prometheus-alertmanager` | 2 | UP |
| `monitoring-kube-prometheus-apiserver` | 1 | UP |
| `monitoring-kube-prometheus-coredns` | 2 | UP |
| `monitoring-kube-prometheus-kubelet` | 2 | UP |
| `monitoring-kube-prometheus-operator` | 1 | UP |
| `monitoring-kube-prometheus-prometheus` | 1 | UP |
| `monitoring-kube-state-metrics` | 1 | UP |
| `monitoring-prometheus-node-exporter` | 2 | UP |

### Check targets via the API

```
curl -s http://192.168.30.12:8081/api/v1/targets | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
for t in data['data']['activeTargets']:
    print(t['health'], t['labels'].get('job',''), t['scrapeUrl'])
"
```

### Check the backend target specifically

```
curl -s http://192.168.30.12:8081/api/v1/targets | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
targets = [t for t in data['data']['activeTargets'] if 'humor' in str(t)]
for t in targets:
    print('Health:', t['health'])
    print('URL:', t['scrapeUrl'])
    print('Last error:', t['lastError'])
    print('Last scrape:', t['lastScrape'])
"
```

Expected output:

```
Health: up
URL: http://10.42.1.70:3001/metrics
Last error:
Last scrape: 2026-06-23T11:20:01.061594693Z
```

---

## Available Metrics

### What the backend exposes

The backend uses the default Node.js `prom-client` instrumentation. It exposes process and runtime metrics — no custom application metrics are instrumented in the current implementation.

Confirm available metrics:

```
kubectl exec -n humor-memory-game deploy/backend -- \
  curl -s http://localhost:3001/metrics | grep "^# HELP"
```

### Backend metrics reference

| Metric | Type | Description |
|---|---|---|
| `process_cpu_user_seconds_total` | Counter | Total user CPU time |
| `process_cpu_system_seconds_total` | Counter | Total system CPU time |
| `process_cpu_seconds_total` | Counter | Total CPU time (user + system) |
| `process_resident_memory_bytes` | Gauge | Physical memory in use |
| `process_virtual_memory_bytes` | Gauge | Virtual memory size |
| `process_heap_bytes` | Gauge | V8 heap size |
| `process_open_fds` | Gauge | Open file descriptors |
| `process_start_time_seconds` | Gauge | Process start time (Unix epoch) |
| `nodejs_eventloop_lag_seconds` | Gauge | Event loop lag |
| `nodejs_eventloop_lag_p50_seconds` | Gauge | Event loop lag 50th percentile |
| `nodejs_eventloop_lag_p90_seconds` | Gauge | Event loop lag 90th percentile |
| `nodejs_eventloop_lag_p99_seconds` | Gauge | Event loop lag 99th percentile |
| `nodejs_heap_size_total_bytes` | Gauge | V8 heap total size |
| `nodejs_heap_size_used_bytes` | Gauge | V8 heap used size |
| `nodejs_active_handles_total` | Gauge | Active libuv handles |
| `nodejs_active_requests_total` | Gauge | Active libuv requests |
| `nodejs_version_info` | Gauge | Node.js version label |

> **Note:** Custom application metrics such as `http_requests_total`, `active_games_current`, and `database_connections_current` are not currently instrumented in the backend. Adding these requires modifying the backend source code to register custom `prom-client` metrics. This is a future enhancement — see the Future Enhancements section at the end of this document.

### Cluster metrics reference

The `kube-prometheus-stack` provides a rich set of cluster-level metrics via kube-state-metrics and node-exporter. Key metrics for this cluster:

| Metric | Description |
|---|---|
| `kube_pod_status_phase` | Pod phase (Running, Pending, Failed) |
| `kube_deployment_status_replicas_available` | Available replicas per deployment |
| `kube_node_status_condition` | Node health conditions |
| `container_cpu_usage_seconds_total` | Per-container CPU usage |
| `container_memory_usage_bytes` | Per-container memory usage |
| `node_memory_MemAvailable_bytes` | Available node memory |
| `node_cpu_seconds_total` | Node CPU time by mode |
| `kubelet_running_pods` | Pods running on each node |

---

## PromQL Query Reference

### Using the Prometheus UI

1. Open `http://192.168.30.12:8081`
2. Click the **Query** tab
3. Enter a query in the expression bar
4. Click **Execute**
5. Toggle between **Table** (current values) and **Graph** (time series) views

### Using the API

All queries can also be run via the Prometheus HTTP API:

```
curl -s "http://192.168.30.12:8081/api/v1/query?query=<QUERY>" | \
  python3 -c "import sys,json; data=json.load(sys.stdin); [print(r['metric'], r['value']) for r in data['data']['result']]"
```

---

### Application health queries

**Backend process uptime:**

```
time() - process_start_time_seconds{namespace="humor-memory-game"}
```

Returns seconds since the backend process started.

**Backend CPU usage rate (5 minute average):**

```
rate(process_cpu_seconds_total{namespace="humor-memory-game"}[5m]) * 100
```

Returns CPU usage as a percentage.

**Backend memory usage:**

```
process_resident_memory_bytes{namespace="humor-memory-game"}
```

Returns physical memory in bytes.

**Node.js event loop lag:**

```
nodejs_eventloop_lag_seconds{namespace="humor-memory-game"}
```

Returns the current event loop lag in seconds. Values above 0.1 seconds indicate the process is under load.

**Node.js heap usage:**

```
nodejs_heap_size_used_bytes{namespace="humor-memory-game"} / nodejs_heap_size_total_bytes{namespace="humor-memory-game"} * 100
```

Returns heap utilisation as a percentage.

---

### Kubernetes cluster queries

**All pods currently running in the application namespace:**

```
kube_pod_status_phase{namespace="humor-memory-game", phase="Running"}
```

**Pod restart count:**

```
kube_pod_container_status_restarts_total{namespace="humor-memory-game"}
```

**Available replicas per deployment:**

```
kube_deployment_status_replicas_available{namespace="humor-memory-game"}
```

**Node memory usage percentage:**

```
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

**Node CPU usage percentage:**

```
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Container CPU usage by pod:**

```
rate(container_cpu_usage_seconds_total{namespace="humor-memory-game", container!=""}[5m]) * 100
```

**Container memory usage by pod:**

```
container_memory_usage_bytes{namespace="humor-memory-game", container!=""}
```

---

### Prometheus health queries

**All scrape targets and their health:**

```
up
```

Returns `1` for every target that is reachable, `0` for unreachable. All targets should return `1`.

**Scrape duration by job:**

```
scrape_duration_seconds
```

**Number of time series per job:**

```
scrape_samples_scraped
```

---

### Useful exploration queries

**List all available metric names:**

```
curl -s 'http://192.168.30.12:8081/api/v1/label/__name__/values' | \
  python3 -c "import sys,json; [print(m) for m in json.load(sys.stdin)['data']]" | sort
```

**List all metrics from the backend:**

```
curl -s 'http://192.168.30.12:8081/api/v1/label/__name__/values' | \
  python3 -c "import sys,json; [print(m) for m in json.load(sys.stdin)['data'] if 'node' in m or 'process' in m]" | sort
```

**Check a specific metric exists:**

```
curl -s "http://192.168.30.12:8081/api/v1/query?query=process_resident_memory_bytes" | \
  python3 -c "import sys,json; data=json.load(sys.stdin); print('Results:', len(data['data']['result']))"
```

---

## Verify the ServiceMonitor is Working

The ServiceMonitor tells the Prometheus Operator to scrape the backend. Confirm it is correctly configured and Prometheus is acting on it:

**Check the ServiceMonitor exists:**

```
kubectl get servicemonitor humor-game-monitor -n monitoring -o yaml
```

**Check Prometheus has picked up the scrape config:**

```
curl -s http://192.168.30.12:8081/api/v1/targets | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
pools = data['data']['activeTargets']
humor = [t for t in pools if 'humor' in str(t.get('labels', {}))]
print('Backend targets found:', len(humor))
for t in humor:
    print('  Health:', t['health'])
    print('  URL:', t['scrapeUrl'])
"
```

Expected output:

```
Backend targets found: 1
  Health: up
  URL: http://10.42.1.70:3001/metrics
```

If `Backend targets found: 0`, the ServiceMonitor is not matching any targets. Refer to `MONITORING_TROUBLESHOOTING.md`.

---

## Future Enhancements

The backend currently exposes only default Node.js process metrics. Adding custom application metrics requires modifying the backend source code in `sjchristie/humor-memory-game`.

Metrics that would be valuable to add:

| Metric | Type | Description |
|---|---|---|
| `http_requests_total` | Counter | Total HTTP requests by method, path, status |
| `http_request_duration_seconds` | Histogram | Request duration — enables p50, p95, p99 queries |
| `http_errors_total` | Counter | Total HTTP errors by path |
| `active_games_current` | Gauge | Current active game sessions |
| `game_scores_total` | Counter | Total game scores by difficulty |
| `database_connections_current` | Gauge | Active database connections |

These would be registered using `prom-client` in the backend Express application and exposed via the existing `/metrics` endpoint. Once instrumented, no changes to the Prometheus or ServiceMonitor configuration would be required — Prometheus would pick them up automatically on the next scrape.

---

## Next Steps

| Document | Contents |
|---|---|
| `MONITORING_TROUBLESHOOTING.md` | Ingress routing issues, SSH tunnel errors, RBAC 403, datasource issues, ServiceMonitor no targets |
| `MONITORING_CUSTOM_DASHBOARDS.md` | Building and customising dashboards from scratch |
