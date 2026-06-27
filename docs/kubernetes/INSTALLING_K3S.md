# Document 4 — Installing K3s

**Phase 4 — Kubernetes on Raspberry Pi**  
**Version:** 1.0  

---

## Overview

This document covers the installation of K3s on both Raspberry Pi nodes, configuration of `sre-mgmnt-01` to manage the cluster remotely, setup of the Synology CSI driver for persistent iSCSI storage, and a verification test to confirm the cluster is fully operational.

K3s is installed manually because the worker join depends on the control plane being confirmed healthy first — something that cannot be reliably automated across two independent boot sequences.

> **Prerequisites:** Both nodes must be fully prepared and verified as per Document 3 before starting here.

---

## 4.1 Why K3s Instead of Full Kubernetes

| Aspect | Full Kubernetes | K3s |
|--------|----------------|-----|
| Install time | 45–60 minutes | 2–5 minutes |
| Control plane RAM | 1.5GB | 400MB |
| Binary size | 500MB+ | 50MB |
| Setup complexity | 15+ manual steps | 1 curl command |
| kubectl commands | All work | All work (identical) |
| YAML manifests | Standard | Identical — no changes needed |
| ARM64 support | Works but more config | Native, first class |
| Maintenance | Manual component upgrades | Automatic |
| Certification | CNCF | CNCF (same) |
| Home lab fit | Overkill | Excellent |

K3s is real, certified Kubernetes. Everything you learn here transfers directly to full Kubernetes. The only difference is lighter internal tooling.

K3s includes out of the box:

- **CoreDNS** — DNS for service discovery
- **Flannel** — pod networking
- **local-path-provisioner** — storage provisioner
- **metrics-server** — performance metrics

> **Note:** Traefik ingress is disabled during installation (`--disable traefik`). The application deployment in Document 5 uses a NodePort service for external access instead.

---

## 4.2 Install K3s on cp-01 (Control Plane)

SSH into cp-01:

```bash
ssh cp-01
```

Install K3s with Traefik disabled and the node IP explicitly set:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable traefik --node-ip=192.168.30.20" sh -
```

Wait approximately 30 seconds, then confirm the control plane is Ready:

```bash
sudo kubectl get nodes
```

Expected output:

```
NAME    STATUS   ROLES           AGE   VERSION
cp-01   Ready    control-plane   12s   v1.35.5+k3s1
```

Confirm the K3s service is running:

```bash
sudo systemctl status k3s --no-pager | head -5
```

Expected output:

```
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-06-05 21:15:48 AEST; 1min 28s ago
 Invocation: 9cdd61b6ac1c41f99eb009f20c061dfd
       Docs: https://k3s.io
```

> Do not proceed to the worker until STATUS shows **Ready**.

---

## 4.3 Retrieve the Node Token

The node token is required to join the worker node to the cluster. Copy it to a readable location on cp-01:

```bash
sudo cp /var/lib/rancher/k3s/server/node-token /tmp/node-token

sudo chmod 644 /tmp/node-token
```

Open a new terminal on `sre-mgmnt-01` and copy the token locally:

```bash
scp -i ~/.ssh/pi_key kubeadmin@192.168.30.20:/tmp/node-token ~/k3s-node-token

cat ~/k3s-node-token
```

Keep this terminal open — you will need the token value in the next section.

---

## 4.4 Install K3s on wrk-01 (Worker)

SSH into wrk-01:

```bash
ssh wrk-01
```

Install the K3s agent — replace `<PASTE_TOKEN_HERE>` with the contents of `~/k3s-node-token`:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL="https://192.168.30.20:6443" \
  K3S_TOKEN="<PASTE_TOKEN_HERE>" \
  sh -
```

---

## 4.5 Confirm Both Nodes Are Ready

SSH into cp-01:

```bash
ssh cp-01
```

Check both nodes are showing Ready:

```bash
sudo kubectl get nodes
```

Expected output:

```
NAME     STATUS   ROLES           AGE    VERSION
cp-01    Ready    control-plane   18m    v1.35.5+k3s1
wrk-01   Ready    <none>          2m6s   v1.35.5+k3s1
```

Run a final cluster check:

```bash
sudo kubectl get nodes -o wide

sudo kubectl get pods -A
```

Both nodes showing **Ready** confirms the cluster is operational:

- **cp-01** — control plane, Pi 5, `192.168.30.20` ✅
- **wrk-01** — worker, Pi 4, `192.168.30.21` ✅
- **CoreDNS** — running ✅
- **local-path-provisioner** — running ✅
- **metrics-server** — running ✅

---

## 4.6 Configure sre-mgmnt-01 to Access the Cluster

All cluster management is done from `sre-mgmnt-01`. Configure `kubectl` access here so you never need to SSH into `cp-01` to run cluster commands.

Copy the SSH key to `sre-mgmnt-01` if not already present:

```bash
scp -i ~/.ssh/pi_key ~/.ssh/pi_key sre@192.168.30.12:~/.ssh/pi_key

scp -i ~/.ssh/pi_key ~/.ssh/pi_key.pub sre@192.168.30.12:~/.ssh/pi_key.pub
```

SSH into `sre-mgmnt-01`:

```bash
ssh sre-mgmnt-01
```

Copy the kubeconfig from `cp-01` and update the server address from localhost to the Pi's IP:

```bash
mkdir -p ~/.kube

# 1. Create a copy you own (this will prompt cleanly for your password)
ssh -t -i ~/.ssh/pi_key kubeadmin@192.168.30.20 "sudo cp /etc/rancher/k3s/k3s.yaml ~/k3s-tmp.yaml && sudo chown kubeadmin:kubeadmin ~/k3s-tmp.yaml"

# Note: will get a "message Connection to 192.168.30.20 closed."

# 2. Pull the file locally and apply the IP replacement (No sudo/password needed here)
ssh -i ~/.ssh/pi_key kubeadmin@192.168.30.20 "cat ~/k3s-tmp.yaml" | sed 's/127.0.0.1/192.168.30.20/g' > ~/.kube/k3s-config.yaml

# 3. Clean up the temporary file on the remote host
ssh -i ~/.ssh/pi_key kubeadmin@192.168.30.20 "rm ~/k3s-tmp.yaml"
```

Make the file executable
```
chmod 600 ~/.kube/k3s-config.yaml
```

Make this the default kubeconfig:

```bash
echo 'export KUBECONFIG=~/.kube/k3s-config.yaml' >> ~/.bashrc

source ~/.bashrc
```

Test cluster access from `sre-mgmnt-01`:

```bash
kubectl get nodes

kubectl get pods -A
```

Expected result
```bash
NAME     STATUS   ROLES           AGE   VERSION
cp-01    Ready    control-plane   9d    v1.35.5+k3s1
wrk-01   Ready    <none>          9d    v1.35.5+k3s1
```

`kubectl` is now fully configured on `sre-mgmnt-01`. All remaining Phase 4 commands are run from here.

To assign the **worker** role to your `wrk-01` node, you need to apply the specific Kubernetes label `node-role.kubernetes.io/worker`.

In Kubernetes, roles are not hardcoded names; they are driven entirely by labels matching that exact prefix.

Run this command from your terminal on `sre-mgmt-01`:

```bash
kubectl label node wrk-01 node-role.kubernetes.io/worker=
```

Verify the Change

Run your node check again to see the updated output:

```bash
kubectl get nodes
```

Expected output
```
NAME     STATUS   ROLES           AGE     VERSION
cp-01    Ready    control-plane   9m54s   v1.35.5+k3s1
wrk-01   Ready    worker          6m52s   v1.35.5+k3s1
```

---

## 4.7 Verification Test — Deploy Nginx

Deploy a test workload to confirm cross-node pod scheduling and networking are working correctly. Run all commands from `sre-mgmnt-01`:

```bash
kubectl create deployment nginx --image=nginx:alpine

kubectl expose deployment nginx --port=80 --type=NodePort

kubectl get svc nginx
```

```bash
kubectl get pods
```

Expect result
```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6658f59756-j85s2   1/1     Running   0          115s
```

Note the NodePort assigned, then curl the service using `cp-01`'s IP:

```bash
curl http://192.168.30.20:<NodePort>
```

Expected result:
```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>

.....

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Scale to 2 replicas and verify pods spread across both nodes:

```bash
kubectl scale deployment nginx --replicas=2

kubectl get pods -o wide
```

Expected results:

- nginx serving traffic on cp-01 ✅
- 2 replicas spread across both nodes ✅
- Pod on cp-01 (`10.42.0.x`) ✅
- Pod on wrk-01 (`10.42.1.x`) ✅
- Cross-node pod networking working ✅

```
NAME                     READY   STATUS    RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
nginx-6658f59756-j85s2   1/1     Running   0          3m22s   10.42.1.26   wrk-01   <none>           <none>
nginx-6658f59756-nd2kq   1/1     Running   0          8s      10.42.0.5    cp-01    <none>           <none>
```

Clean up the test deployment:

```bash
kubectl delete deployment nginx

kubectl delete svc nginx
```

---

## 4.8 Synology CSI Driver Setup

### Why Use the Synology CSI Driver

The core reason is **pod mobility** — the ability to move a pod from one node to another without losing its data.

**Without the CSI driver (local-path):**

When K3s provisions storage using `local-path`, the data lives physically on whichever node the pod is running on. If that pod needs to move to the other node — because the first node crashed, needs maintenance, or Kubernetes decided to reschedule it — the data cannot move with it. The pod either starts with no data or cannot start at all because it is tied to a specific node.

**With the CSI driver (Synology iSCSI):**

The data lives on NAS2, completely separate from both nodes. When a pod needs to move from `cp-01` to `wrk-01`, the CSI driver detaches the iSCSI LUN from `cp-01` and reattaches it to `wrk-01`. The pod starts on `wrk-01` with all its data intact. Neither node owns the storage — NAS2 does.

**Additional benefits:**

- NAS2 has RAID 1 — persistent volume data is mirrored across two drives automatically
- Storage grows independently of Pi hardware — add more drives to NAS2 without touching the cluster
- LUN snapshots can be taken directly in DSM outside of Kubernetes
- 4.5TB available on NAS2 versus ~50GB on Pi SD cards

---

### 4.8.1 Install open-iscsi on Both Nodes

The iSCSI initiator must be installed on both nodes before the CSI driver can attach LUNs.

**cp-01:**
```bash
ssh cp-01
```

```bash
sudo apt install -y open-iscsi

sudo systemctl enable iscsid --now

sudo systemctl status iscsid --no-pager | head -3

exit
```



**wrk-01:**

```
ssh wrk-01
```

```bash
sudo apt install -y open-iscsi

sudo systemctl enable iscsid --now

sudo systemctl status iscsid --no-pager | head -3

exit
```

---

### 4.8.2 Configure NAS2

Log into DSM on NAS2 and install **SAN Manager** from Package Center — search for "SAN Manager" and install it. This enables iSCSI on the Synology.

Create a dedicated DSM user for the CSI driver:

1. Go to DSM → **Control Panel** → **User & Group**
2. Click **Create**
3. Fill in:
   - Name: `k3s-csi`
   - Password: set a strong password eg ({CPjHF9 and record it — you will need it in Section 4.8.4
   - Add to **administrators** group — tick administrators and leave users ticked, then click Next
4. Assign shared folder permissions — leave all blank. The CSI driver manages iSCSI LUNs directly and does not need access to shared folders
5. Assign application permissions — allow **DSM** access only, deny everything else
6. Leave speed limit defaults
7. Click **Done**


---

### 4.8.3 Install Helm

**What is Helm?**

Helm is the package manager for Kubernetes — the equivalent of `apt` on Debian or `pacman` on Arch Linux. Instead of manually writing and applying individual YAML manifest files, Helm packages them together into a reusable unit called a **chart**. A chart contains all the manifests, default configuration values, and templating logic needed to deploy an application or tool to Kubernetes.

**Why Helm for the CSI driver?**

The Synology CSI driver requires multiple Kubernetes resources to be deployed in the correct order — ServiceAccounts, ClusterRoles, ClusterRoleBindings, Deployments, and DaemonSets across multiple namespaces. Managing these individually with `kubectl apply` would require careful sequencing and is error-prone. The CSI driver is distributed as a Helm chart so the entire deployment is handled in a single command with consistent, tested configuration.

> **Note:** In this project Helm is used only for the Synology CSI driver. The application manifests and monitoring stack are deployed directly with `kubectl apply` using the YAML files produced in Documents 5 and 6.

Install Helm on `sre-mgmnt-01`:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm version
```

Expected output:

```
helm installed into /usr/local/bin/helm

version.BuildInfo{Version:"v3.21.0", ...}
```

---

### 4.8.4 Deploy the Synology CSI Driver

Clone the Synology CSI driver repository on `sre-mgmnt-01`:

```bash
git clone https://github.com/SynologyOpenSource/synology-csi.git

cd synology-csi
```

Create the client config file from the template:

```bash
cp config/client-info-template.yml config/client-info.yml
```

Edit the config file with NAS2 credentials:

```bash
vim config/client-info.yml
```

Replace the contents with the following — substituting `<your-k3s-csi-password>` with the password set in Section 4.8.2:

```yaml
clients:
  - host: <your nas ip>
    port: 5001
    https: true
    username: k3s-csi
    password: <your-k3s-csi-password>
```

Verify the file:

```bash
cat config/client-info.yml
```

Ensure the KUBECONFIG environment variable is set:

```bash
echo 'export KUBECONFIG=~/.kube/k3s-config.yaml' >> ~/.bashrc

source ~/.bashrc
```

Create the namespace and deploy the CSI driver:

```bash
kubectl create namespace synology-csi

kubectl create secret generic client-info-secret \
  --from-file=client-info.yml=config/client-info.yml \
  --namespace synology-csi

kubectl apply -f deploy/kubernetes/v1.20/
```

Check the pods are running:

```bash
kubectl get pods -n synology-csi
```

Expected output:

```
NAME                        READY   STATUS    RESTARTS   AGE
synology-csi-controller-0   4/4     Running   0          44s
synology-csi-node-jtqkq     2/2     Running   0          44s
synology-csi-node-zdzfx     2/2     Running   0          44s
```

Check the storage class was created:

```bash
kubectl get storageclass
```

Expected output:

```
NAME                     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)     rancher.io/local-path   Delete          WaitForFirstConsumer   false                  79m
synology-iscsi-storage   csi.san.synology.com    Retain          Immediate              true                   97s
```

---

### 4.8.5 Create a Second Storage Class with Delete Policy

Two storage classes are needed — one that retains data when a PVC is deleted (for databases), and one that automatically cleans up (for ephemeral workloads).

| Policy | What Happens When PVC is Deleted |
|--------|----------------------------------|
| **Retain** | LUN stays on NAS2, PV goes to Released state, data preserved |
| **Delete** | LUN is automatically deleted from NAS2, space reclaimed |

The `synology-iscsi-storage` class installed by the CSI driver uses `Retain`. Create a second class with `Delete`:

Annotated Version — Read First

```yaml
# StorageClass — synology-iscsi-delete
# Provisioner: Synology CSI driver
# Reclaim policy: Delete — LUN is automatically removed from NAS2 when PVC is deleted
# Use for: ephemeral workloads where automatic cleanup is preferred

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: synology-iscsi-delete        # Name used in PVC storageClassName field
provisioner: csi.san.synology.com    # Synology CSI driver provisioner
parameters:
  fsType: ext4                       # Filesystem type formatted on the LUN
  dsm: '192.168.0.7'                 # NAS2 IP address
  location: '/volume1'               # Volume on NAS2 where LUNs are created
reclaimPolicy: Delete                # LUN deleted automatically when PVC is removed
allowVolumeExpansion: true           # Allows PVC storage to be increased after creation
```

Production Copy — Run This Command

```bash
kubectl apply -f - << 'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: synology-iscsi-delete
provisioner: csi.san.synology.com
parameters:
  fsType: ext4
  dsm: '192.168.0.7'
  location: '/volume1'
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF
```

Verify both storage classes are present:

```bash
kubectl get storageclass
```

Expected output:

```
NAME                     PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path (default)     rancher.io/local-path   Delete          WaitForFirstConsumer   false                  82m
synology-iscsi-delete    csi.san.synology.com    Delete          Immediate              true                   5s
synology-iscsi-storage   csi.san.synology.com    Retain          Immediate              true                   3m
```

Storage class reference going forward:

- `synology-iscsi-storage` — Retain, for databases and important persistent data
- `synology-iscsi-delete` — Delete, for ephemeral workloads
- `local-path` — default K3s local storage, node-bound

---

### 4.8.6 Test the CSI Driver

Create a test PVC to confirm the CSI driver can provision a LUN on NAS2:

```bash
kubectl apply -f - << 'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: synology-iscsi-storage
  resources:
    requests:
      storage: 1Gi
EOF
```

Check that it binds:

```bash
kubectl get pvc test-pvc
```

Expected output:

```
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS             AGE
test-pvc   Bound    pvc-df1db351-a381-4ed1-a70d-c7243a6168e0   1Gi        RWO            synology-iscsi-storage   11s
```

The PVC is **Bound** — the CSI driver successfully provisioned a 1Gi iSCSI LUN on NAS2.

Confirm the LUN appeared in NAS2 SAN Manager


then delete the test PVC:

```bash
kubectl delete pvc test-pvc
```

Check the PV status:

```bash
kubectl get pv
```

Expect result
```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM              STORAGECLASS             VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-0e2ae8f7-c5b2-4a7f-aa3f-386e3dcc6a90   1Gi        RWO            Retain           Released   default/test-pvc   synology-iscsi-storage   <unset>                          3m56s
```

Delete the PV from Kubernetes

Run this command to remove the volume entry from your cluster configuration:


```
kubectl delete pv <VOLUMEATTRIBUTESCLASS>
```

The LUN remains on NAS2 because the storage class uses `reclaimPolicy: Retain`. The PV moves to a `Released` state. To fully remove it, delete the LUN manually from NAS2 SAN Manager.

> **Note:** This retain behaviour is intentional for the `synology-iscsi-storage` class. Use `synology-iscsi-delete` for workloads where automatic cleanup is preferred.

---

## 4.9 Verification Checklist

- [ ] K3s installed on `cp-01` — STATUS shows Ready
- [ ] Node token retrieved and saved to `~/k3s-node-token` on `sre-mgmnt-01`
- [ ] K3s agent installed on `wrk-01`
- [ ] Both nodes show Ready in `kubectl get nodes`
- [ ] CoreDNS, local-path-provisioner, and metrics-server running
- [ ] `kubectl` configured on `sre-mgmnt-01` — `kubectl get nodes` works without SSH to `cp-01`
- [ ] KUBECONFIG exported in `~/.bashrc`
- [ ] Nginx test deployment confirmed cross-node pod scheduling and networking
- [ ] `open-iscsi` installed and `iscsid` running on both nodes
- [ ] NAS2 SAN Manager installed and `k3s-csi` DSM user created
- [ ] Helm installed on `sre-mgmnt-01`
- [ ] Synology CSI driver deployed — all pods Running
- [ ] Both storage classes present — `synology-iscsi-storage` and `synology-iscsi-delete`
- [ ] Test PVC bound successfully and confirmed in NAS2 SAN Manager

---

*Next: Document 5 — Deploying the Application*
