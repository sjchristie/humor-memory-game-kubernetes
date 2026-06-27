# MONITORING_TROUBLESHOOTING

## Monitoring Stack Troubleshooting

This document covers operational issues encountered during the Phase 4 Observability implementation. Each issue includes the symptom observed, root cause, and the fix applied. All issues were encountered and resolved on the actual Raspberry Pi K3s cluster — not in a simulated environment.

Issues are grouped by area: dashboard, Prometheus scraping, access, and cluster tooling.

---

## Environment Reference

| Component | Detail |
|---|---|
| Management VM | `sre-mgmt-01` — `192.168.30.12` |
| Control plane | `cp-01` — `192.168.30.20` |
| Worker node | `wrk-01` — `192.168.30.21` |
| Application namespace | `humor-memory-game` |
| Monitoring namespace | `monitoring` |
| Grafana | `http://192.168.30.12:3000` |
| Prometheus | `http://192.168.30.12:8081` |

---

## Dashboard Issues

---

### ISSUE-MON-001 — Dashboard panels show "Datasource not found"

**Symptom:**

Dashboard loads in Grafana but all panels display:

```
Datasource PBFA97CFB590B2093 was not found
```

or:

```
Datasource ${DS_PROMETHEUS} was not found
```

**Root cause:**

The dashboard JSON file contains a hardcoded datasource UID (`PBFA97CFB590B2093`) from a different Grafana environment. When loaded via ConfigMap provisioning, `${DS_PROMETHEUS}` is not resolved — it is a Grafana import-time variable only resolved during manual UI import. Neither value matches the Prometheus datasource UID in this installation.

The `kube-prometheus-stack` chart assigns the Prometheus datasource the UID `prometheus`. The dashboard JSON must reference this exact value.

**Confirmed fix:**

Replace all datasource UIDs in the dashboard JSON file with `prometheus`:

```
sed -i 's/PBFA97CFB590B2093/prometheus/g' monitoring/dashboards/comprehensive-dashboard.json
sed -i 's/\${DS_PROMETHEUS}/prometheus/g' monitoring/dashboards/comprehensive-dashboard.json
```

Verify:

```
grep '"uid"' monitoring/dashboards/comprehensive-dashboard.json | grep -v "Grafana"
```

Expected output:

```
        "uid": "prometheus"
        "uid": "prometheus"
        "uid": "prometheus"
```

Delete and recreate the ConfigMap with the corrected file:

```
kubectl delete configmap comprehensive-game-dashboard -n monitoring

kubectl create configmap comprehensive-game-dashboard \
  --from-file=comprehensive-dashboard.json=monitoring/dashboards/comprehensive-dashboard.json \
  --namespace monitoring

kubectl label configmap comprehensive-game-dashboard -n monitoring grafana_dashboard=1
```

Wait 60 seconds for the sidecar to reload, then refresh Grafana.

**Prevention:**

Always verify the datasource UID in dashboard JSON files before creating the ConfigMap. The correct UID for this installation is `prometheus`. Retrieve the actual UID at any time with:

```
kubectl exec -n monitoring deploy/monitoring-grafana -- \
  curl -s -u admin:admin123 http://localhost:3000/api/datasources | \
  python3 -c "import sys,json; ds=json.load(sys.stdin); [print(d['uid'], d['name']) for d in ds]"
```

---

### ISSUE-MON-002 — Dashboard home page shows Welcome screen instead of dashboard

**Symptom:**

After login, Grafana shows the "Welcome to Grafana" onboarding screen instead of the Comprehensive Game Monitoring Dashboard.

**Root cause:**

Two possible causes:

1. The `default_home_dashboard_path` in `monitoring-values.yaml` is set to an incorrect path
2. The Grafana organisation home dashboard preference has not been set on the running instance

**Confirmed fix — wrong path:**

The `kube-prometheus-stack` sidecar writes dashboards to `/tmp/dashboards/` — not `/var/lib/grafana/dashboards/default/`. The values file must reference the correct path.

Confirm the correct path in `monitoring-values.yaml`:

```
grep "default_home_dashboard_path" monitoring/monitoring-values.yaml
```

Expected output:

```
      default_home_dashboard_path: /tmp/dashboards/comprehensive-dashboard.json
```

If the path is incorrect, update the file and run `helm upgrade`:

```
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f monitoring/monitoring-values.yaml
```

After the upgrade, restart the port-forward:

```
pkill -f "kubectl port-forward"

kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring --address=0.0.0.0 > /dev/null 2>&1 &
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 8081:9090 -n monitoring --address=0.0.0.0 > /dev/null 2>&1 &
```

**Confirmed fix — running instance:**

Apply the home dashboard preference via the Grafana API without redeployment:

```
kubectl exec -n monitoring deploy/monitoring-grafana -- \
  curl -s -u admin:admin123 http://localhost:3000/api/search | \
  python3 -c "
import sys, json
results = json.load(sys.stdin)
for r in results:
    if 'Comprehensive' in r.get('title', ''):
        print(r['uid'], r['title'])
"
```

Then set the home dashboard using the UID returned:

```
kubectl exec -n monitoring deploy/monitoring-grafana -- \
  curl -s -X PUT -u admin:admin123 \
  -H "Content-Type: application/json" \
  -d '{"homeDashboardUID":"<UID>"}' \
  http://localhost:3000/api/org/preferences
```

Expected output:

```
{"message":"Preferences updated"}
```

---

### ISSUE-MON-003 — Grafana sidecar logs show only "Loading incluster config..."

**Symptom:**

```
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard
```

Output shows only repeated entries:

```
{"time": "2026-06-23T06:14:41.852449+00:00", "level": "INFO", "msg": "Loading incluster config..."}
{"time": "2026-06-23T06:15:41.876477+00:00", "level": "INFO", "msg": "Loading incluster config..."}
```

**Root cause:**

This is normal sidecar behaviour — not an error. The sidecar polls the Kubernetes API every 60 seconds. `Loading incluster config...` appears on each poll cycle. The sidecar only writes additional log lines when it detects and loads new or changed ConfigMaps.

**How to verify the dashboard actually loaded:**

Do not rely on the sidecar logs to confirm dashboard loading. Check the sidecar write folder directly:

```
kubectl exec -n monitoring deploy/monitoring-grafana -c grafana-sc-dashboard -- \
  ls -la /tmp/dashboards/
```

Look for `comprehensive-dashboard.json` in the output. If present, the dashboard has been loaded into Grafana successfully.

Also confirm the ConfigMap label is correct:

```
kubectl get configmap comprehensive-game-dashboard -n monitoring -o jsonpath='{.metadata.labels}'
```

Expected output:

```
{"grafana_dashboard":"1"}
```

---

## Prometheus Scraping Issues

---

### ISSUE-MON-004 — ServiceMonitor shows 0/0 up — No active targets

**Symptom:**

Prometheus targets page at `http://192.168.30.12:8081/targets` shows:

```
serviceMonitor/monitoring/humor-game-monitor/0    0/0 up    No active targets
```

**Root cause — missing namespaceSelector:**

The ServiceMonitor is deployed in the `monitoring` namespace but the backend runs in the `humor-memory-game` namespace. Without a `namespaceSelector`, the Prometheus Operator only discovers services within the same namespace as the ServiceMonitor.

**Root cause — missing service label:**

The backend Service (`manifests/11-backend-service.yaml`) had no `metadata.labels` block. The ServiceMonitor selector `matchLabels: app: backend` matches against Service labels — not the Service's internal pod selector. Without the label on the Service itself, the ServiceMonitor finds no matching Service.

**Root cause — unnamed service port:**

The ServiceMonitor references ports by name. The original backend Service had an unnamed port (`3001/TCP`). The ServiceMonitor port field `port: metrics` requires a corresponding named port on the Service.

**Confirmed fix — ServiceMonitor:**

The correct `monitoring/humor-service-monitor.yaml`:

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

**Confirmed fix — backend Service:**

The correct `manifests/11-backend-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: humor-memory-game
  labels:
    app: backend
spec:
  selector:
    app: backend
  ports:
  - name: metrics
    port: 3001
    targetPort: 3001
    protocol: TCP
  type: ClusterIP
```

Apply both:

```
kubectl apply -f manifests/11-backend-service.yaml
kubectl apply -f monitoring/humor-service-monitor.yaml
```

Wait 30 seconds then verify:

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
"
```

Expected output:

```
Health: up
URL: http://10.42.1.70:3001/metrics
Last error:
```

---

### ISSUE-MON-005 — Prometheus RBAC 403 on node metrics endpoints

**Symptom:**

Prometheus targets page shows node targets with errors:

```
kubernetes-nodes:     0/2 UP    Error: server returned HTTP status 403 Forbidden
kubernetes-cadvisor:  0/2 UP    Error: server returned HTTP status 403 Forbidden
```

**Root cause:**

K3s requires `nonResourceURLs` permissions in the Prometheus ClusterRole for node and cAdvisor metric endpoints. Without `/metrics` and `/metrics/cadvisor` in the RBAC rules, all node-level scrape requests return HTTP 403.

**Confirmed fix:**

The ClusterRole must include:

```yaml
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
```

Delete and reapply the RBAC:

```
kubectl delete clusterrole prometheus
kubectl delete clusterrolebinding prometheus
kubectl apply -f monitoring/prometheus-rbac.yaml
```

Then restart the Prometheus pod to pick up the new permissions:

```
kubectl rollout restart deployment/monitoring-kube-prometheus-operator -n monitoring
```

Wait for the operator to reconcile, then check the targets page.

---

### ISSUE-MON-006 — Dashboard panels show "No data" after scraping confirmed working

**Symptom:**

Prometheus targets page shows the backend as `UP` and scraping successfully, but Grafana dashboard panels all show "No data".

**Root cause:**

The dashboard JSON was built expecting custom application metrics (`http_requests_total`, `active_games_current`, `database_connections_current`) that are not instrumented in the backend. The backend only exposes default Node.js `prom-client` metrics.

**Resolution:**

The dashboard panels querying non-existent metrics will always show "No data". This is expected behaviour — not a configuration error.

Panels that do show data with the current backend instrumentation:
- Application Health
- HTTP Request Rate (from health probe and Prometheus scrape traffic)
- Response Time Percentiles
- Memory Usage
- CPU Usage
- Database Connections (from kube-state-metrics)

Adding custom metrics requires modifying the backend source code. See the Future Enhancements section in `MONITORING_METRICS.md`.

---

## Access Issues

---

### ISSUE-MON-007 — Ingress returns 404 when accessing via IP address

**Symptom:**

Browser shows Nginx 404 when navigating to `http://127.0.0.1:8080` or `http://192.168.30.12:3000` via SSH tunnel.

**Root cause:**

The Nginx Ingress controller routes traffic based on the HTTP `Host` header. When accessing via an IP address, the browser sends the IP as the host header. The Ingress has no rule matching an IP address — only `grafana.gameapp.local` — so Nginx returns 404.

**Resolution:**

Use port-forwarding (Option B in `MONITORING_SETUP.md`) instead of Ingress for IP-based access. Port-forwarding bypasses the Ingress controller entirely and connects directly to the service.

If Ingress access is required without modifying the local hosts file, add a catch-all rule to the Ingress resource:

```
kubectl patch ingress monitoring-grafana -n monitoring --type='json' \
  -p='[{"op": "add", "path": "/spec/rules/-", "value": {"http": {"paths": [{"backend": {"service": {"name": "monitoring-grafana", "port": {"number": 80}}}, "path": "/", "pathType": "ImplementationSpecific"}]}}}]'
```

> **Note:** This catch-all approach is not recommended for production environments. It exposes the dashboard to any traffic regardless of hostname. Use port-forwarding for development access.

---

### ISSUE-MON-008 — SSH tunnel returns "Connection refused"

**Symptom:**

SSH tunnel connects successfully but browser shows `ERR_CONNECTION_RESET` and the terminal shows:

```
channel 3: open failed: connect failed: Connection refused
channel 4: open failed: connect failed: Connection refused
```

**Root cause:**

The SSH tunnel command `ssh -L 8080:localhost:8080 sre@192.168.30.12` forwards local port 8080 to port 8080 on the management VM. Nothing was listening on port 8080 on the management VM — the Ingress controller listens on port 80 of the cluster nodes, not on the management VM.

**Confirmed fix:**

For Ingress-based access, the correct tunnel command targets the cluster node directly through the management VM as a jump host:

```
ssh -L 8080:192.168.30.20:80 sre@192.168.30.12
```

For port-forward-based access, no SSH tunnel is needed — port-forward with `--address=0.0.0.0` makes the service directly accessible from any machine on the network at `http://192.168.30.12:3000`.

---

### ISSUE-MON-009 — Port-forward stops working after helm upgrade

**Symptom:**

After running `helm upgrade`, the Grafana or Prometheus port-forward returns connection errors or no longer responds.

**Root cause:**

`helm upgrade` restarts the affected pods. The port-forward process is bound to a specific pod instance — when the pod restarts, the port-forward loses its connection and stops working.

**Confirmed fix:**

After any `helm upgrade`, kill all active port-forwards and restart them:

```
pkill -f "kubectl port-forward"
```

Wait for the new pods to be ready:

```
kubectl get pods -n monitoring
```

Then restart the port-forwards:

```
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring --address=0.0.0.0 > /dev/null 2>&1 &

kubectl port-forward svc/monitoring-kube-prometheus-prometheus 8081:9090 -n monitoring --address=0.0.0.0 > /dev/null 2>&1 &
```

---

## Cluster Tooling Issues

---

### ISSUE-MON-010 — kubectl edit fails with "unable to launch the editor vi"

**Symptom:**

```
kubectl edit ingress monitoring-grafana -n monitoring
error: unable to launch the editor "vi"
```

**Root cause:**

`vi` is not installed on `sre-mgmt-01`. `kubectl edit` requires an interactive text editor.

**Confirmed fix:**

Use `kubectl patch` instead of `kubectl edit` for live resource modifications. `kubectl patch` applies changes directly without requiring an editor:

```
kubectl patch ingress monitoring-grafana -n monitoring --type='json' \
  -p='[{"op": "add", "path": "/spec/rules/-", "value": {...}}]'
```

For file-based changes, edit the source YAML file directly using `cat > file << EOF` heredoc syntax and reapply with `kubectl apply -f`.

If an interactive editor is needed, set `nano` as the editor for the session:

```
KUBE_EDITOR=nano kubectl edit ingress monitoring-grafana -n monitoring
```

---

### ISSUE-MON-011 — wget not available in backend container

**Symptom:**

```
kubectl exec -n humor-memory-game deploy/backend -- wget -qO- http://localhost:3001/metrics
wget: can't connect to remote host: Connection refused
command terminated with exit code 1
```

**Root cause:**

`wget` is not installed in the backend container image. The error message `Connection refused` is misleading — the port is listening correctly but `wget` is failing to execute.

**Confirmed fix:**

Use `curl` instead. `curl` is available in the backend container:

```
kubectl exec -n humor-memory-game deploy/backend -- \
  curl -s http://localhost:3001/metrics | head -20
```

When testing connectivity from inside a pod, always try `curl` first. `wget` availability depends on the base image used — Node.js images typically include `curl` but not `wget`.

---

## Quick Reference — Diagnostic Commands

| Goal | Command |
|---|---|
| Check all monitoring pods | `kubectl get pods -n monitoring` |
| Check Prometheus targets | `curl -s http://192.168.30.12:8081/api/v1/targets \| python3 -c "import sys,json; [print(t['health'], t['labels'].get('job',''), t['scrapeUrl']) for t in json.load(sys.stdin)['data']['activeTargets']]"` |
| Check backend scrape target | `curl -s http://192.168.30.12:8081/api/v1/targets \| python3 -c "import sys,json; [print(t['health'], t['scrapeUrl'], t['lastError']) for t in json.load(sys.stdin)['data']['activeTargets'] if 'humor' in str(t)]"` |
| Check sidecar dashboard folder | `kubectl exec -n monitoring deploy/monitoring-grafana -c grafana-sc-dashboard -- ls -la /tmp/dashboards/` |
| Check Grafana datasources | `kubectl exec -n monitoring deploy/monitoring-grafana -- curl -s -u admin:admin123 http://localhost:3000/api/datasources \| python3 -c "import sys,json; [print(d['uid'], d['name']) for d in json.load(sys.stdin)]"` |
| Check ConfigMap labels | `kubectl get configmap comprehensive-game-dashboard -n monitoring -o jsonpath='{.metadata.labels}'` |
| Check backend metrics endpoint | `kubectl exec -n humor-memory-game deploy/backend -- curl -s http://localhost:3001/metrics \| head -10` |
| Check backend service labels | `kubectl get svc backend -n humor-memory-game --show-labels` |
| Check active port-forwards | `ps aux \| grep "kubectl port-forward"` |
| Restart all port-forwards | `pkill -f "kubectl port-forward"` |

---

## Next Steps

| Document | Contents |
|---|---|
| `MONITORING_CUSTOM_DASHBOARDS.md` | Building and customising dashboards from scratch |
