# MONITORING_CUSTOM_DASHBOARDS

## Custom Dashboard Building Guide

This document covers building custom Grafana dashboards from scratch for the Humor Memory Game monitoring stack. It covers panel types, PromQL query construction, variables, annotations, thresholds, design patterns, and how to make dashboards persistent.

This is an advanced reference document. The monitoring stack must be installed and accessible before following any steps here. For importing pre-built dashboards or loading dashboards via ConfigMap, see `MONITORING_DASHBOARDS.md`.

---

## Environment Reference

| Component | Detail |
|---|---|
| Grafana | `http://192.168.30.12:3000` |
| Prometheus | `http://192.168.30.12:8081` |
| Grafana credentials | `admin` / `admin123` |
| Application namespace | `humor-memory-game` |
| Dashboard files | `monitoring/dashboards/` |

---

## Before Building a Dashboard

### Always verify queries in Prometheus first

Before adding a query to a Grafana panel, test it in the Prometheus UI at `http://192.168.30.12:8081`. A query that returns no results in Prometheus will return no results in Grafana — the problem is always in the query or the metric, not in Grafana.

### Know what metrics are available

The backend exposes default Node.js `prom-client` metrics only. Custom application metrics such as `http_requests_total` and `game_scores_total` are not instrumented. Queries using these metric names will return no data.

Confirmed available metrics for this environment:

```
process_cpu_seconds_total
process_resident_memory_bytes
process_virtual_memory_bytes
nodejs_eventloop_lag_seconds
nodejs_heap_size_used_bytes
nodejs_heap_size_total_bytes
kube_pod_status_phase
kube_deployment_status_replicas_available
container_cpu_usage_seconds_total
container_memory_usage_bytes
node_memory_MemAvailable_bytes
node_cpu_seconds_total
up
```

To list all available metrics:

```
curl -s 'http://192.168.30.12:8081/api/v1/label/__name__/values' | \
  python3 -c "import sys,json; [print(m) for m in sorted(json.load(sys.stdin)['data'])]"
```

---

## Creating a New Dashboard

1. Open Grafana at `http://192.168.30.12:3000`
2. Click **Dashboards** in the left sidebar
3. Click **New** → **New Dashboard**
4. Click **Add visualisation**
5. Select **Prometheus** as the data source

The panel editor opens with a query editor at the bottom and the visualisation preview at the top.

---

## Panel Types

### Stat panel

Use for single current values — health status, counts, or key performance indicators. The value is displayed large with optional colour thresholds.

Good for: pod count, uptime, current memory usage, connection count.

Example query:

```
kube_pod_status_phase{namespace="humor-memory-game", phase="Running"}
```

Panel settings: set **Calculation** to `Last (not null)` to show the most recent value rather than a sum or mean.

### Time series panel

Use for metrics that change over time — the standard line graph. Shows trends and patterns across the selected time range.

Good for: CPU usage over time, memory trends, request rates, event loop lag.

Example query:

```
rate(process_cpu_seconds_total{namespace="humor-memory-game"}[5m]) * 100
```

Panel settings: set the **Unit** to match the metric — `percent (0-100)` for CPU, `bytes` for memory.

### Gauge panel

Use for metrics expressed as a percentage or within a known range. Displays a dial with configurable min, max, and threshold zones.

Good for: heap utilisation, memory pressure, CPU saturation.

Example query:

```
nodejs_heap_size_used_bytes{namespace="humor-memory-game"} / nodejs_heap_size_total_bytes{namespace="humor-memory-game"} * 100
```

Panel settings: set **Min** to `0`, **Max** to `100`, and configure thresholds at 70 (warning) and 90 (critical).

### Heatmap panel

Use for distribution data — showing how values are distributed across time. Most useful for response time histograms when histogram metrics are available.

Good for: request duration distributions, event loop lag variance.

Example query (requires histogram metrics):

```
rate(http_request_duration_seconds_bucket[5m])
```

> **Note:** Heatmap panels require histogram metrics (`_bucket` suffix). These are only available if custom application metrics are instrumented in the backend.

### Table panel

Use for displaying multiple metrics or multiple instances in a structured format. Columns can be sorted and filtered.

Good for: pod status across all deployments, resource usage by container.

Example query:

```
kube_pod_container_status_restarts_total{namespace="humor-memory-game"}
```

Panel settings: use **Transform** → **Labels to fields** to convert metric labels into table columns.

---

## Building a Panel Step by Step

### 1. Enter the query

In the query editor at the bottom of the panel editor, enter the PromQL query. Click outside the field or press Shift+Enter to execute.

### 2. Set the legend

The legend label controls how each series is identified in the panel. Use label references in double curly braces:

```
{{pod}} — {{container}}
```

This displays the pod name and container name for each series.

### 3. Choose the visualisation

Click the visualisation dropdown in the top right of the panel editor. Select the panel type that suits the data — time series for trends, stat for current values, gauge for ranges.

### 4. Set the unit

In the right panel under **Standard options → Unit**, set the unit to match the metric. Common units:

| Metric type | Unit setting |
|---|---|
| CPU percentage | `Percent (0-100)` |
| Memory bytes | `bytes (IEC)` |
| Duration seconds | `seconds (s)` |
| Plain count | `short` |
| Requests per second | `reqps` |

### 5. Configure thresholds

Under **Standard options → Thresholds**, add colour thresholds to make the panel immediately readable:

| Colour | Meaning | Typical threshold |
|---|---|---|
| Green | Healthy | Base colour |
| Yellow | Warning | 70% of limit |
| Red | Critical | 90% of limit |

### 6. Apply and save

Click **Apply** to close the panel editor. Click the **Save dashboard** icon in the top right. Give the dashboard a name and click **Save**.

---

## Advanced Customisation

### Variables — dynamic filters

Variables make dashboards reusable across namespaces, pods, or nodes without editing queries. They appear as dropdown filters at the top of the dashboard.

To add a variable:

1. Open **Dashboard Settings** (gear icon, top right)
2. Click **Variables** → **Add variable**
3. Configure:
   - **Name:** `namespace`
   - **Type:** `Query`
   - **Data source:** `Prometheus`
   - **Query:** `label_values(up, namespace)`
   - **Refresh:** `On dashboard load`
4. Click **Apply** then **Save dashboard**

Use the variable in panel queries with a `$` prefix:

```
kube_pod_status_phase{namespace="$namespace"}
```

Useful variables for this environment:

| Variable name | Query | Purpose |
|---|---|---|
| `namespace` | `label_values(kube_pod_info, namespace)` | Filter by namespace |
| `pod` | `label_values(kube_pod_info{namespace="$namespace"}, pod)` | Filter by pod |
| `node` | `label_values(kube_node_info, node)` | Filter by cluster node |

### Annotations — event markers

Annotations mark specific events on time series panels — useful for correlating deployments or restarts with metric changes.

To add an annotation:

1. Open **Dashboard Settings** → **Annotations**
2. Click **Add annotation query**
3. Configure:
   - **Name:** `Pod restarts`
   - **Data source:** `Prometheus`
   - **Query:** `changes(kube_pod_container_status_restarts_total{namespace="humor-memory-game"}[1m]) > 0`
4. Click **Apply** then **Save dashboard**

Pod restart events will appear as vertical markers on all time series panels.

### Thresholds and colour overrides

Thresholds apply colours based on the current value. Set them per panel under **Standard options → Thresholds**.

For the application health panel showing `up{job="backend"}`:

| Value | Colour | Meaning |
|---|---|---|
| 0 | Red | Service down |
| 1 | Green | Service up |

For event loop lag showing `nodejs_eventloop_lag_seconds`:

| Value | Colour | Meaning |
|---|---|---|
| 0 | Green | No lag |
| 0.1 | Yellow | Moderate lag |
| 0.5 | Red | High lag — process under pressure |

### Panel links

Panels can link to other dashboards or external URLs. Useful for creating a top-level overview dashboard that links to detailed views.

In panel settings under **Panel links**, add a link to the Prometheus query for the same metric:

- **Title:** `View in Prometheus`
- **URL:** `http://192.168.30.12:8081/graph?g0.expr=<query>`

---

## Dashboard Layout Best Practices

### Row structure

Organise panels into logical rows using the **Add row** option. Each row can be collapsed for cleaner navigation on large dashboards.

Recommended structure for this application:

| Row | Content | Panel types |
|---|---|---|
| Application health | Pod status, uptime, restart count | Stat |
| Resource usage | CPU, memory, heap | Time series, Gauge |
| Performance | Event loop lag, response times | Time series |
| Infrastructure | Node CPU, node memory | Time series |

### Panel sizing

Grafana uses a 24-column grid. Consistent panel widths make dashboards easier to read:

| Use case | Width | Height |
|---|---|---|
| Full-width time series | 24 | 8 |
| Half-width time series | 12 | 8 |
| Stat panel (key metric) | 6 | 4 |
| Gauge panel | 4 | 6 |

### Refresh rates

Set the dashboard refresh rate based on how time-sensitive the data is:

| Use case | Refresh rate |
|---|---|
| Active incident monitoring | 5s |
| Live operational view | 30s |
| Standard monitoring | 1m |
| Historical analysis | Off |

The refresh rate is set in the top right of the dashboard next to the time range picker.

---

## Exporting and Persisting Dashboards

Dashboards built in the Grafana UI are stored in Grafana's internal SQLite database. They do not survive if the Grafana pod is deleted or the monitoring namespace is removed. To make a dashboard persistent it must be exported as JSON and stored in the repository.

### Export a dashboard as JSON

1. Open the dashboard in Grafana
2. Click the **Share** icon (top right, next to Edit)
3. Click **Export**
4. Enable **Export for sharing externally**
5. Click **Save to file**

The JSON file downloads to your local machine.

### Commit to the repository

Copy the exported JSON to `monitoring/dashboards/` in the repository and commit it:

```
# On sre-mgmt-01, after copying the file from your local machine
git add monitoring/dashboards/my-custom-dashboard.json
git commit -m "Add custom monitoring dashboard"
git push
```

### Fix the datasource UID before committing

Exported dashboard JSON may contain a hardcoded datasource UID. Replace it with the environment-agnostic value before committing:

```
sed -i 's/<YOUR_GRAFANA_UID>/prometheus/g' monitoring/dashboards/my-custom-dashboard.json
```

Verify:

```
grep '"uid"' monitoring/dashboards/my-custom-dashboard.json | grep -v "Grafana"
```

All datasource UID values should show `prometheus`.

### Load via ConfigMap

Once committed, load the dashboard via ConfigMap following the steps in `MONITORING_DASHBOARDS.md` Option B. This makes the dashboard available automatically on any future deployment.

---

## Troubleshooting Custom Dashboards

### Panel shows "No data"

Work through in order:

1. Test the query in Prometheus first at `http://192.168.30.12:8081`
2. If Prometheus returns no data, the metric does not exist or the label filters are wrong
3. Check the metric name is correct — run the full metrics list command from the Before You Begin section
4. Check the namespace label — use `humor-memory-game` not `humor-game`
5. Check the time range — ensure data exists for the selected window

### Panel shows "Datasource not found"

The dashboard JSON references a datasource UID that does not exist in this Grafana instance. See ISSUE-MON-001 in `MONITORING_TROUBLESHOOTING.md`.

### Variable dropdown shows no options

The variable query is not returning results. Test the variable query directly in Prometheus. Confirm the label referenced in `label_values()` exists on the metrics being queried.

### Dashboard not appearing after ConfigMap load

Check the sidecar folder:

```
kubectl exec -n monitoring deploy/monitoring-grafana -c grafana-sc-dashboard -- \
  ls -la /tmp/dashboards/
```

If the file is not present, verify the ConfigMap label is `grafana_dashboard=1`. See ISSUE-MON-003 in `MONITORING_TROUBLESHOOTING.md`.

---

## Next Steps

| Document | Contents |
|---|---|
| `MONITORING_METRICS.md` | Full PromQL query reference and available metrics |
| `MONITORING_TROUBLESHOOTING.md` | Datasource issues, sidecar problems, scrape target failures |
