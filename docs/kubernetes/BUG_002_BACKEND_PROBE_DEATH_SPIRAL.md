# Bug Report — BUG-002: Backend Probe Death Spiral

| Field | Detail |
|-------|--------|
| ID | BUG-002 |
| Title | Backend pods enter CrashLoopBackOff due to aggressive health probe exhausting database connections |
| Severity | High |
| Status | Resolved |
| Reported | 2026-06-14 |
| Environment | k3s cluster — humor-memory-game namespace |
| Reporter | Stephen Christie |

---

## Summary

Both backend pods entered a continuous restart cycle. The liveness and readiness probes were hitting `/api/health` at high frequency, each probe triggering a database connection from the pool. Under this load the connection pool was exhausted, causing the health check itself to fail, which triggered Kubernetes to restart the pod, which started the cycle again.

This is known as a probe death spiral — the mechanism designed to detect failures was itself causing them.

---

## Symptom

```bash
kubectl get pods -l app=backend -n humor-memory-game
```

```
NAME                       READY   STATUS    RESTARTS        AGE
backend-5db4db948d-h88lf   1/1     Running   15 (105s ago)   96m
backend-5db4db948d-vjkz8   1/1     Running   15 (4m ago)     96m
```

Get Logs from a Previously Crashed Pod Instance

If a pod _was_ running earlier but restarted or crashed, you can pull the logs of that dead container instance using the **`-p` (previous)** flag

```
kubectl logs redis-7f96659d49-tgrmz -n humor-memory-game -p
```


Both pods showing 15 restarts within 96 minutes. Backend logs showed graceful shutdown being triggered repeatedly:

```
🔌 Database client removed from pool

🛑 Received shutdown signal. Starting graceful shutdown...
🔌 HTTP server closed.
🔌 Database connection pool closed
🗄️  Database connections closed.
🔌 Redis: Connection ended
🔌 Redis connection closed
🔗 Redis connection closed.
✅ Graceful shutdown completed. Goodbye! 👋
npm notice
npm notice New major version of npm available! 10.9.8 -> 11.17.0
npm notice Changelog: https://github.com/npm/cli/releases/tag/v11.17.0
npm notice To update run: npm install -g npm@11.17.0
npm notice
```

---

## Investigation

### Step 1 — Identified the restart pattern

The pods were not crashing due to an application error — they were being deliberately terminated by Kubernetes after the liveness probe failed. The graceful shutdown sequence in the logs confirmed the pod was receiving a SIGTERM from Kubernetes, not crashing.

### Step 2 — Reviewed the probe configuration

Checked `10-backend-deployment.yaml`:

```yaml
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
```

The readiness probe was checking every 5 seconds with only a 5 second initial delay. With 2 replicas, this generates a probe request every 2.5 seconds on average across the deployment.

### Step 3 — Identified the root cause

The `/api/health` endpoint performs a live database connectivity check on every call — it acquires a connection from the pool to verify the database is reachable. Under normal load this is fine. Under probe load at this frequency, the health checks were competing with application requests for connections, exhausting the pool.

When the pool was exhausted, the next health check timed out, the liveness probe registered a failure, and Kubernetes terminated the pod. On restart the cycle began again immediately.

---

## Root Cause

The liveness and readiness probes were both configured to hit `/api/health` at high frequency. The `/api/health` endpoint acquires a database connection on every call. With a 5 second readiness probe period across 2 replicas, the probes were generating a continuous stream of database connection requests that competed with and eventually exhausted the connection pool.

The liveness probe is designed to detect a dead process. The readiness probe is designed to gate traffic until the pod is ready. Neither should be generating database load at this frequency in steady state.

---

## Fix

Update probe timing in `10-backend-deployment.yaml` to reduce database connection pressure while maintaining meaningful health checking:

```yaml
livenessProbe:
  httpGet:
    path: /api/health
    port: 3001
  initialDelaySeconds: 45
  periodSeconds: 30
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /api/health
    port: 3001
  initialDelaySeconds: 20
  periodSeconds: 15
  failureThreshold: 3
```

**Why these values:**

| Setting | Old Value | New Value | Reason |
|---------|-----------|-----------|--------|
| liveness `initialDelaySeconds` | 30 | 45 | Give the application more time to establish connection pools before first check |
| liveness `periodSeconds` | 10 | 30 | Liveness detects dead processes — 30s is sufficient, reduces DB load by 66% |
| readiness `initialDelaySeconds` | 5 | 20 | Wait longer before routing traffic — avoids probe load during startup |
| readiness `periodSeconds` | 5 | 15 | Reduces readiness probe DB connections by 66% in steady state |
| `failureThreshold` | not set (default 3) | 3 | Explicit — gives the application a buffer before Kubernetes acts |

> **Note:** If restarts continue after applying these probe values, the `/api/health` endpoint itself needs to be reviewed. A health check that leaks connections or blocks under load defeats the purpose of the probe. The correct long-term solution is a health endpoint that performs a lightweight ping rather than a full connection acquisition.

---

## GitOps Deployment Process

This fix is a Kubernetes manifest change only — no application code changes and no new Docker image required. It is deployed through the existing GitOps workflow.

### Step 1 — Apply the fix to the manifest

On `sre-mgmnt-01`, edit `10-backend-deployment.yaml`:

```bash
cd ~/humor-memory-game-kubernetes/manifests
vim 10-backend-deployment.yaml
```

Update the `livenessProbe` and `readinessProbe` blocks as shown in the Fix section above.

---
### Step 2 — Test the file

```bash
kubectl apply -f 10-backend-deployment.yaml --dry-run=server
```

#### What this does:

- **`-f 10-backend-deployment.yaml`**: This explicitly tells `kubectl` to only look at this one file.

- **`--dry-run=server`**: Kubernetes will send this deployment definition to the API server, which will parse the YAML, check the `livenessProbe` and `readinessProbe` definitions, validate the syntax, and check if the resources it references (like ConfigMaps or PVCs) exist.

- **No changes**: Since it is a dry run, no pods will be restarted, and no changes will be made to your cluster.

#### How to interpret the result:

1. **If it says `deployment.apps/backend-deployment configured (dry run)`**: Your YAML syntax is correct, and the configuration is valid according to your cluster's schema. You are safe to deploy.
    
2. **If it shows an error**: It will highlight the exact line and the reason for the failure (e.g., `unknown field`, `invalid value`, or `missing required field`).

Confirm no YAML errors before committing.

---
### Step 3 — Deploy the change

Since you have already validated the file with the `dry-run` command, you simply remove the `--dry-run` flag:

```
kubectl apply -f 10-backend-deployment.yaml
```

After running the command, Kubernetes will start rolling out the changes. You can track the progress with these commands:

```bash
kubectl get pods -l app=backend -n humor-memory-game
```

Check the Rollout Status:

```bash
kubectl rollout status deployment/backend -n humor-memory-game
```

- **Verify the Probe Changes:** If you want to confirm your new probes are actually active, you can inspect the running pod's YAML:
  
```bash
# Replace <pod-name> with the actual name from 'kubectl get pods'
kubectl get pod backend-548f5b777-k7psp -n humor-memory-game -o yaml | grep -A 10 livenessProbe
```

---
### Step 4 — Verify restarts stop

```bash
kubectl get pods -l app=backend -n humor-memory-game
```

The `RESTARTS` count should stop climbing. Monitor for 10 minutes to confirm stability.

```bash
kubectl logs -n humor-memory-game deployment/backend --tail=20
```

Logs should show normal request handling with no shutdown signals.

---

### Step 5 — Commit and push to GitHub

```bash
cd ~/humor-memory-game-kubernetes

git status

git add manifests/10-backend-deployment.yaml

git commit -m "fix: BUG-002 reduce backend probe frequency to prevent connection pool exhaustion
- liveness initialDelaySeconds: 30 -> 45
- liveness periodSeconds: 10 -> 30
- readiness initialDelaySeconds: 5 -> 20
- readiness periodSeconds: 5 -> 15
- failureThreshold: 3 explicit on both probes"

git push origin main
```

---
## Why This Did Not Occur in Docker Compose

Docker Compose does not have liveness or readiness probes. The `healthcheck:` directive in Docker Compose is advisory — it marks a container as unhealthy but does not restart it unless `restart: on-failure` is configured and the health check itself exits non-zero.

In Kubernetes, probes are enforced by the kubelet. A failed liveness probe results in immediate pod termination. A failed readiness probe removes the pod from the Service endpoint list. The enforcement is automatic and aggressive by design — Kubernetes assumes the probe configuration is correct and acts on failures immediately.

The behaviour that caused this bug is Kubernetes working as designed. The fix is configuring probes that reflect the actual health characteristics of the application rather than the defaults.

---

## Notes

- Both liveness and readiness probes should use the same endpoint in this application since there is no separate lightweight health check path. The fix is reducing frequency, not changing the endpoint.
- In a future phase, consider splitting the health check into two endpoints — a lightweight `/api/ping` for the liveness probe (TCP only, no DB) and `/api/health` for the readiness probe (full connectivity check). This follows the Kubernetes probe design intent — liveness for dead process detection, readiness for traffic gating.
- This fix is deployed via GitOps — the change is version-controlled, reviewable, and reproducible.
