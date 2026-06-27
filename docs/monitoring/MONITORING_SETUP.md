# MONITORING_SETUP

## Prometheus & Grafana Monitoring Stack

This document covers the complete end-to-end installation of the observability stack for the Humor Memory Game Kubernetes deployment. It uses the `kube-prometheus-stack` Helm chart — the Prometheus Operator pattern — which replaces the manual manifest-based approach used in Phase 3 with a single, managed installation.

Follow this document from start to finish. There are no branches or jumps to other documents during installation.

This is a Phase 4 Observability document in the Homelab to Production portfolio series.

---

## Environment Reference

| Component | Detail |
|---|---|
| Management VM | `sre-mgmt-01` — `192.168.30.12` |
| Control plane node | `cp-01` — `192.168.30.20` — Raspberry Pi 5 |
| Worker node | `wrk-01` — `192.168.30.21` — Raspberry Pi 4 |
| Repository | `humor-memory-game-kubernetes` |
| Monitoring directory | `/home/sre/humor-memory-game-kubernetes/monitoring/` |
| Grafana access | `http://192.168.30.12:3000` |
| Prometheus access | `http://192.168.30.12:8081` |
| Grafana credentials | `admin` / `admin123` |

All commands in this document are run from `sre-mgmt-01` unless stated otherwise.

---

## Prerequisites

### Confirm the repository is present

The monitoring directory will be created inside the existing Kubernetes infrastructure repository. Confirm it is present on the management VM:

```bash
cd /home/sre/humor-memory-game-kubernetes
```

```
pwd
```

Expected output:
```
/home/sre/humor-memory-game-kubernetes
```

### Create the monitoring directory structure

All configuration files, scripts, and dashboard JSON files for this phase are stored under `monitoring/`. Create the directory structure now:

```bash
mkdir -p monitoring/dashboards
```

Verify:
```
ls -la monitoring/
```

Expected output:
```
drwxr-xr-x 1 sre sre   18 Jun 23 10:00 .
drwxr-xr-x 1 sre sre  144 Jun 23 10:00 ..
drwxr-xr-x 1 sre sre    0 Jun 23 10:00 dashboards
```

All subsequent commands assume `/home/sre/humor-memory-game-kubernetes/` as the working directory unless stated otherwise.

---

## Step 1: Install ingress-nginx

The `monitoring-values.yaml` references `ingressClassName: nginx`. The nginx IngressClass must exist in the cluster before the Prometheus stack is installed — if it does not, the Ingress objects are created in a broken pending state and require manual intervention to recover.

### Add the Helm repository

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

Expected output:
```
"ingress-nginx" has been added to your repositories
```

Expected output:
```
"ingress-nginx" has been added to your repositories
```

```bash
helm repo update
```

Expected output:
```
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
Update Complete. ⎈Happy Helming!⎈
```

### Install ingress-nginx

```bash
helm install ingress-nginx ingress-nginx/ingress-nginx -n kube-system
```

Expected output:
```
NAME: ingress-nginx
LAST DEPLOYED: Mon Jun 22 15:56:13 2026
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
...
```

### Verify the IngressClass exists

```bash
kubectl get ingressclass
```

Expected output:
```
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       117s
```

Do not proceed until `nginx` appears in this output.

### Verify the ingress-nginx controller service

```bash
kubectl get svc -n kube-system ingress-nginx-controller
```

Expected output:
```
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP                   PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   10.43.241.34   192.168.30.20,192.168.30.21   80:31800/TCP,443:31427/TCP   2m23s
```

---

## Step 2: Prepare the monitoring namespace

### Remove any existing monitoring namespace

If a `monitoring` namespace exists from a previous installation, remove it completely before proceeding. This avoids resource conflicts with the Helm install.

```bash
kubectl delete namespace monitoring
```

Namespace deletion is asynchronous. Wait until the following command confirms it is gone before continuing:

```bash
kubectl get namespace monitoring
```

Expected result:
```
Error from server (NotFound): namespaces "monitoring" not found
```

> **Note:** Do not proceed until this command returns `NotFound`. Applying resources into a terminating namespace will fail silently.


### Create the namespace

```bash
kubectl create namespace monitoring
```

Expected result:
```
namespace/monitoring created
```

---

## Step 3: Configure RBAC

Prometheus requires cluster-wide read permissions to discover and scrape metrics from nodes, pods, and services. This step creates the ServiceAccount, ClusterRole, and ClusterRoleBinding that provide those permissions.

The three components work together as follows:

- **ServiceAccount** — the identity the Prometheus pod runs as inside the cluster
- **ClusterRole** — defines what that identity is allowed to read across all namespaces
- **ClusterRoleBinding** — connects the identity to the permissions

Save the following as `monitoring/prometheus-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs:
      - get
      - list
      - watch
  - apiGroups: ["networking.k8s.io"]
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - nonResourceURLs:
      - /metrics
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
```

Apply:
```bash
kubectl apply -f monitoring/prometheus-rbac.yaml
```

Expected output:
```
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
serviceaccount/prometheus created
```

> ⚠️ **Flag for implementation review**
> The `kube-prometheus-stack` Helm chart creates its own ServiceAccount, ClusterRole, and ClusterRoleBinding automatically during install. This manual RBAC may be redundant or could conflict with chart-managed RBAC. Verify at implementation whether this step is required for your environment.

---

## Step 4: Pre-load the dashboard ConfigMap

The `kube-prometheus-stack` Helm chart deploys a Grafana sidecar container that watches for ConfigMaps labelled `grafana_dashboard=1` and loads them into Grafana automatically. Creating the ConfigMap before the Helm install means the dashboard is available the moment Grafana starts — no manual import required.

The comprehensive dashboard, 'comprehensive-dashboard.json' is used here as it covers the full application monitoring view across all available metrics.

### Confirm the dashboard file is present

Must display the file '**comprehensive-dashboard.json**'

```bash
find monitoring/dashboards/ -maxdepth 1 -type f -exec basename {} \;
```

Expected output:
```
advanced-custom-dashboard.json
comprehensive-dashboard.json
custom-dashboard.json
```

If the file is not present, the dashboard JSON files have not yet been committed to the repository. Add them to `monitoring/dashboards/` and commit before continuing. The dashboard JSON files are sourced from the `monitoring/dashboards/` directory of the `humor-memory-game-kubernetes` repository — they are not generated during installation.

### Verify the dashboard datasource UID

The dashboard JSON must have the Prometheus datasource UID set to `prometheus` — the UID assigned by the `kube-prometheus-stack` chart to its Prometheus datasource. If the file contains `${DS_PROMETHEUS}` or a hardcoded UID from another environment, the dashboard will load but all panels will show "Datasource not found".

The file in `monitoring/dashboards/comprehensive-dashboard.json` has been corrected and committed to the repository.

The grep command filters on `"uid"` but the dashboard JSON also contains the dashboard's own UID field (`"uid": "comprehensive-game-dashboard"`). Use the following command to show only datasource UIDs:

Verify before creating the ConfigMap:
```bash
grep '"uid"' monitoring/dashboards/comprehensive-dashboard.json | grep -v "Grafana" | grep -v "comprehensive"
```

Expected output — all datasource UIDs showing `prometheus`:

```
        "uid": "prometheus"
        "uid": "prometheus"
        "uid": "prometheus"
        ...
```


If any line shows `${DS_PROMETHEUS}` or a hardcoded UID from another environment, replace all datasource UIDs with `prometheus`:

```bash
sed -i 's/\${DS_PROMETHEUS}/prometheus/g' monitoring/dashboards/comprehensive-dashboard.json
```

Verify again before continuing.


### Create the ConfigMap

```bash
kubectl create configmap comprehensive-game-dashboard \
  --from-file=comprehensive-dashboard.json=monitoring/dashboards/comprehensive-dashboard.json \
  --namespace monitoring
```

Expected output:
```
configmap/comprehensive-game-dashboard created
```

### Apply the sidecar label

The Grafana sidecar only detects ConfigMaps that carry this specific label:

```bash
kubectl label configmap comprehensive-game-dashboard -n monitoring grafana_dashboard=1
```

Expected output:
```
configmap/comprehensive-game-dashboard labeled
```

### Verify the ConfigMap contains data

```bash
kubectl get configmap comprehensive-game-dashboard -n monitoring -o yaml | grep -c "panels"
```

Expected output:
```
1
```

NOTE: Only execute the following command id the return value from above is not 1.
This must return a value greater than `0`. If it returns `0`, the JSON was not embedded correctly. Delete the ConfigMap and repeat from the beginning of this step:

```bash
kubectl delete configmap comprehensive-game-dashboard -n monitoring
```

---

## Step 5: Install the Prometheus Stack

### Prepare the Helm values file

The following values file configures Grafana with a fixed password, Ingress hostnames, and the home dashboard path. The `default_home_dashboard_path` tells Grafana to use the pre-loaded ConfigMap dashboard as the home page instead of the Welcome screen.

Save as `monitoring/monitoring-values.yaml`:

```yaml
grafana:
  adminPassword: "admin123"
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - grafana.gameapp.local
    path: /
  grafana.ini:
    dashboards:
      default_home_dashboard_path: /tmp/dashboards/comprehensive-dashboard.json
  readinessProbe:
    initialDelaySeconds: 30
    timeoutSeconds: 5
    periodSeconds: 10
    failureThreshold: 5
  livenessProbe:
    initialDelaySeconds: 60
    timeoutSeconds: 5
    periodSeconds: 15
    failureThreshold: 10

prometheus:
  prometheusSpec:
    ingress:
      enabled: true
      ingressClassName: nginx
      hosts:
        - prometheus.gameapp.local
      paths:
        - /
```

> **Note:** The `default_home_dashboard_path` must match the folder the Grafana sidecar writes dashboards to. The sidecar is configured by the chart to use `/tmp/dashboards` — confirmed via `kubectl describe deployment monitoring-grafana -n monitoring`. If this path is set incorrectly, Grafana starts but the home dashboard does not load. The correct path was confirmed during implementation by inspecting the sidecar environment variable `FOLDER: /tmp/dashboards`.

> **Note:** `adminPassword` is set here to prevent the chart from generating a random password. To retrieve a random password from an existing installation run:
>
> ```
> kubectl get secret --namespace monitoring monitoring-grafana \
>   -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
> ```

> ⚠️ **Flag for implementation review**
> The Prometheus Ingress is configured under `prometheus.prometheusSpec.ingress`. The standard `kube-prometheus-stack` key path is `prometheus.ingress`. If the Prometheus Ingress resource is not created during the Helm install, update this values file to use `prometheus.ingress` and run `helm upgrade`. Verify at implementation.

### Add the Helm repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Expected output:
```
"prometheus-community" has been added to your repositories
```

```bash
helm repo update
```

Expected output:
```
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
```

### Install the stack

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f monitoring/monitoring-values.yaml
```

Expected output:
```
NAME: monitoring
LAST DEPLOYED: Mon Jun 22 16:39:35 2026
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=monitoring"
...
```

---

## Step 6: Verify the Installation

### Check all pods are running

```bash
kubectl get pods -n monitoring
```

Wait until all pods show `**Running**`. This may take 2–3 minutes on Raspberry Pi hardware.

Expected output:

```
NAME                                                     READY   STATUS
alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running
monitoring-grafana-56ddf4477c-tjf75                      3/3     Running
monitoring-kube-prometheus-operator-748cc88c88-6vgwl     1/1     Running
monitoring-kube-state-metrics-6b8f7fb688-w55kp           1/1     Running
monitoring-prometheus-node-exporter-mfw4w                1/1     Running
monitoring-prometheus-node-exporter-xgpnf                1/1     Running
prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running
```

> ⚠️ **Note:** `monitoring-grafana` shows `3/3` when the sidecar is running alongside the main Grafana container. If it shows `2/3` during startup, wait — the sidecar is still initialising.

### Verify the Ingress resources

```bash
kubectl get ingress -n monitoring
```

Expected output:
```
NAME                 CLASS   HOSTS                   ADDRESS                       PORTS   AGE
monitoring-grafana   nginx   grafana.gameapp.local   192.168.30.20,192.168.30.21   80      3m37s
```

### Verify the dashboard ConfigMap was detected

Check the Grafana sidecar logs to confirm the dashboard was picked up:

```bash
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard
```

What to expect in the logs

- **`Loading incluster config...`** This is **completely normal**. It just means the sidecar is awake and checking for dashboard files. You can safely ignore it.
    
How to fix the "Connection refused" error

If you see `Connection refused at localhost:3000`, it means the sidecar found your dashboards, but it can't hand them over because **Grafana isn't answering the door**.

To fix it, check these three things:

1. **Give it a minute:** Grafana takes longer to boot up than the sidecar. If the error stops after a minute or two, your deployment is fine.
    
2. **Check the port:** Ensure Grafana is actually running on port `3000`. If you changed Grafana's port, the sidecar won't know where to look.
    
3. **Check the sidecar settings:** If Grafana is using a custom setup, make sure the sidecar's environment variables (like `GF_SIDECAR_ENDPOINT`) point to the right URL.

To confirm the dashboard was loaded, check the sidecar folder directly:

```bash
kubectl exec -n monitoring deploy/monitoring-grafana -c grafana-sc-dashboard -- ls -la /tmp/dashboards/comprehensive-dashboard.json
```

Look for `comprehensive-dashboard.json` in the output. If present, the dashboard has been loaded successfully. If absent, verify the ConfigMap label is correct:

```bash
kubectl get configmap comprehensive-game-dashboard -n monitoring -o jsonpath='{.metadata.labels}'
```

Expected output:
```
{"grafana_dashboard":"1"}
```

> **Note:** If this command returns nothing or `Error from server (NotFound)`, the ConfigMap no longer exists — this can happen if the monitoring namespace was deleted and recreated. Return to Step 4 and recreate the ConfigMap before continuing.

---

## Step 7: Configure the ServiceMonitor

The `kube-prometheus-stack` uses the Prometheus Operator pattern. Instead of editing Prometheus scrape configuration directly, a `ServiceMonitor` resource declares how Prometheus should scrape your application. The Operator reads the ServiceMonitor and updates the Prometheus configuration automatically.

Save as `monitoring/humor-service-monitor.yaml`:

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

> **Note:** The `release: monitoring` label is required — the Prometheus Operator uses this label to determine which ServiceMonitors to include.

> **Note:** The `namespaceSelector` is required because the backend runs in the `humor-memory-game` namespace while the ServiceMonitor is deployed in the `monitoring` namespace. Without it, the Prometheus Operator only discovers services within the `monitoring` namespace and will never find the backend. Confirmed during implementation — Prometheus showed `0/0 up` until `namespaceSelector` was added.

> **Note:** The `port` value references the named port `metrics` on the backend Service — not the port number. The ServiceMonitor requires a named port to match against. This requires a corresponding named port on the backend Service manifest — see the backend service note below.

Apply:
```bash
kubectl apply -f monitoring/humor-service-monitor.yaml
```

Expected output:
```
servicemonitor.monitoring.coreos.com/humor-game-monitor created
```

### Verify the ServiceMonitor is recognised

```bash
kubectl get servicemonitors -n monitoring
```

Expected output:
```
NAME                                                 AGE
humor-game-monitor                                   39s
monitoring-grafana                                   5m33s
monitoring-kube-prometheus-alertmanager              5m33s
monitoring-kube-prometheus-apiserver                 5m33s
monitoring-kube-prometheus-coredns                   5m33s
monitoring-kube-prometheus-kube-controller-manager   5m33s
monitoring-kube-prometheus-kube-etcd                 5m33s
monitoring-kube-prometheus-kube-proxy                5m33s
monitoring-kube-prometheus-kube-scheduler            5m33s
monitoring-kube-prometheus-kubelet                   5m33s
monitoring-kube-prometheus-operator                  5m33s
monitoring-kube-prometheus-prometheus                5m33s
monitoring-kube-state-metrics                        5m33s
monitoring-prometheus-node-exporter                  5m33s
```

### Backend service requirements

The ServiceMonitor selector `matchLabels: app: backend` must match a label on the backend Service in the `humor-memory-game` namespace. The backend Service also requires a named port `metrics` for the ServiceMonitor port reference to resolve.

Confirm the backend Service manifest at `manifests/11-backend-service.yaml` matches the following:

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

Two requirements confirmed during implementation:

- `metadata.labels.app: backend` — the Service must carry this label. Without it the ServiceMonitor selector finds no matching Service and Prometheus shows `0/0 up`. The original Phase 3 manifest had no labels on the Service — only a selector inside the spec.
- `ports.name: metrics` — the port must be named. The ServiceMonitor references ports by name not number. The original Phase 3 manifest had an unnamed port `3001/TCP`.

If `manifests/11-backend-service.yaml` does not match the above, update it and apply:

```bash
kubectl apply -f manifests/11-backend-service.yaml
```

---

## Step 8: Access the Monitoring Dashboards

Port-forwarding is the recommended access method for this environment. It requires no DNS changes, no hosts file edits, and no SSH tunnel. The `--address=0.0.0.0` flag makes the forwarded ports accessible from any machine on the same network — including your local laptop browser.

### Set the home dashboard

Before opening the browser, set the home dashboard on the running Grafana instance. The `monitoring-values.yaml` `default_home_dashboard_path` setting applies this automatically on future redeployments — this step applies it to the currently running instance.

Retrieve the dashboard UID:

```bash
kubectl exec -n monitoring deploy/monitoring-grafana -- \
  curl -s -u admin:admin123 http://localhost:3000/api/search | \
  python3 -c "
import sys, json
results = json.load(sys.stdin)
for r in results:
    if 'Comprehensive' in r.get('title', ''):
        print(r['uid'])
"
```

Expected output
```
comprehensive-game-dashboard
```

Set the home dashboard using the UID returned — replace `<UID>` with the value from above:

```bash
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

> **Note:** If the command returns an empty result or the dashboard is not found, the ConfigMap has not been loaded yet. Check the sidecar folder using the command in Step 6 and confirm `comprehensive-dashboard.json` is present before retrying.


### Start port-forwards in the background

Run the following commands on the management VM:

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

Verify both port-forwards are running

Expected output — two processes showing as Running:

```
Starting Monitoring Tunnels... Press Ctrl+C to stop all.
[1] 40849
[2] 40850
Forwarding from 0.0.0.0:3000 -> 3000
Forwarding from 0.0.0.0:8081 -> 9090
```

### Verify access

Open a browser on your local machine and navigate to:

- Grafana: `http://192.168.30.12:3000`
- Prometheus: `http://192.168.30.12:8081`

Log in to Grafana with:

- Username: `admin`
- Password: `admin123`

The Comprehensive Game Monitoring Dashboard should load as the home page. If the Welcome screen appears instead, rerun the Set the Home Dashboard commands above.

> **Note:** Dashboard panels populate automatically within 60 seconds of Grafana starting. Kubernetes liveness and readiness probes hitting `/api/health` every 15–30 seconds, combined with Prometheus scraping `/metrics` every 15 seconds, generate sufficient baseline traffic without any additional steps. No traffic generation script is required.

---

## Step 9: Ingress-Based Access (Alternative)

Port-forwarding is sufficient for development and testing. If you want hostname-based URLs for an extended session or to demonstrate the Kubernetes Ingress pattern, follow this section.

This requires two steps on your local machine. These cannot be avoided — hostname-based Ingress routing requires the browser to send the correct hostname, which requires local DNS resolution.

### Step 9.1 — Add entries to your local machine's hosts file

On your **local machine** (not the management VM):

- **macOS / Linux:** `sudo nano /etc/hosts`
- **Windows:** Open Notepad as Administrator → `C:\Windows\System32\drivers\etc\hosts`

Add to the bottom of the file:

```
127.0.0.1 grafana.gameapp.local
127.0.0.1 prometheus.gameapp.local
```

Save and close.

### Step 9.2 — Open an SSH tunnel

The Ingress controller listens on port 80 of the k3s cluster node. The following command, run on your local machine, forwards your local port 8080 through the management VM to the cluster:

```
ssh -L 8080:192.168.30.20:80 sre@192.168.30.12
```

Keep this terminal open. Closing it drops the tunnel.

### Step 9.3 — Access URLs

- Grafana: `http://grafana.gameapp.local:8080`
- Prometheus: `http://prometheus.gameapp.local:8080`

### Reverting to port-forwarding

To stop using Ingress and return to port-forwarding:

```
kubectl delete ingress -n monitoring --all
```

Close the SSH tunnel, then follow Step 8.

---

## Step 10: Managing Port-Forwards

This section applies when using the port-forwarding access method from Step 8.

### Check active port-forwards

```
ps aux | grep "kubectl port-forward"
```

To see which ports are listening:

```
sudo lsof -i -P -n | grep LISTEN | grep kubectl
```

### Stop all port-forwards

```
pkill -f "kubectl port-forward"
```

### Stop a specific port-forward

```
jobs
```

Kill by job number:

```
kill %1
```

### Quick reference

| Goal | Command |
|---|---|
| List active forwards | `ps aux \| grep "kubectl port-forward"` |
| Kill all forwards | `pkill -f "kubectl port-forward"` |
| Kill specific job | `kill %<job number>` |

---

## Step 11: Teardown

Use the following commands to remove the monitoring stack from the cluster.

Uninstall the Helm release

```bash
helm uninstall monitoring -n monitoring
```

 Remove the ServiceMonitor

```bash
kubectl delete -f monitoring/humor-service-monitor.yaml
```

Remove the dashboard ConfigMap
```bash
kubectl delete configmap comprehensive-game-dashboard -n monitoring
```

Delete the monitoring namespace
```bash
kubectl delete namespace monitoring
```


 Remove an existing Helm repository from your local configuration
```bash
helm repo remove ingress-nginx
```

Remove a repository from your local Helm configuration
```bash
helm repo remove prometheus-community
```

Uninstall the Existing Release in the `kube-system` namespace
```bash
helm uninstall ingress-nginx -n kube-system
```

Remove the security permissions (**RBAC**, or Role-Based Access Control)
```
kubectl delete -f monitoring/prometheus-rbac.yaml
```

> **Note:** Deleting the namespace removes all remaining resources within it including ConfigMaps, Secrets, and Ingress objects. Wait for full termination before reinstalling — use the verification command from Step 2.

---

## Next Steps

| Document | Purpose |
|---|---|
| `MONITORING_DASHBOARDS.md` | Adding dashboards, importing alternatives, managing ConfigMaps |
| `MONITORING_METRICS.md` | PromQL queries, metric exploration, verifying scrape targets |
| `MONITORING_TROUBLESHOOTING.md` | Ingress routing issues, SSH tunnel errors, RBAC 403 |
| `MONITORING_CUSTOM_DASHBOARDS.md` | Building and customising dashboards from scratch |
