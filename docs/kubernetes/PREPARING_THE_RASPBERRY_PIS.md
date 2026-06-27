# Document 3 — Preparing the Raspberry Pis

**Phase 4 — Kubernetes on Raspberry Pi**  
**Version:** 1.0  

---

## Overview

This document covers everything required to get both Raspberry Pi nodes ready for K3s installation. This includes OS selection, flashing, UniFi switch configuration, first boot, static IP configuration, and all kernel-level prerequisites Kubernetes requires.

Complete this document in full for both nodes before proceeding to Document 4 — Installing K3s.

> **Note:** SSH key generation and SSH config are covered in Document 1 — SRE Setup Guide. Ensure those steps are complete before starting here.

---

## 3.1 OS Selection

For a production-ready K3s cluster on Raspberry Pi, the two best operating system choices are **Raspberry Pi OS Lite (64-bit)** or **Ubuntu Server (64-bit)**.

| OS | Best For | Why |
|:---|:---------|:----|
| **Raspberry Pi OS Lite (64-bit)** | Standard and stable setups | The official OS. The Lite version has no desktop, saving maximum RAM and CPU for K3s pods. Best hardware optimisation for the Pi's components and storage controllers. |
| **Ubuntu Server (64-bit)** | Cloud-native parity | Best if you want your Pi cluster to mirror enterprise cloud platforms (AWS, Azure, GCP). Natively supports Cloud-Init and Netplan out of the box. |

**Phase 4 uses Raspberry Pi OS Lite (64-bit) — Bookworm.**

---

## 3.2 Three Mandatory Requirements for K3s on a Pi

Regardless of OS, these three requirements must be met before installing K3s. Skipping any of them will cause the cluster installation to fail.

### Requirement 1 — 64-bit OS (ARM64 / aarch64)

Do not install a 32-bit operating system. Modern containerised workloads and Kubernetes system images are built almost exclusively for 64-bit CPU architectures.

### Requirement 2 — cgroups Enabled

By default, the Raspberry Pi kernel has memory control groups (cgroups) turned off to save memory. Kubernetes requires cgroups to enforce container resource boundaries. This is configured manually in Section 3.9.

### Requirement 3 — Static IP Addresses

Kubernetes nodes communicate constantly using TLS certificates bound to IP addresses. If DHCP changes a node's IP address, the cluster handshake will break. Each Pi must have a static IP before K3s is installed.

---

## 3.3 Node Reference

| Node | Role | IP | User | Hardware |
|------|------|----|------|----------|
| `cp-01` | Control Plane | 192.168.30.20/24 | kubeadmin | Raspberry Pi 5 |
| `wrk-01` | Worker Node | 192.168.30.21/24 | kubeadmin | Raspberry Pi 4 |

---

## 3.4 Flash the OS

Repeat this section for each Pi. Flash `cp-01` first, then `wrk-01`.

### Install Raspberry Pi Imager

On `sre-mgmnt-01`:

```bash
sudo pacman -S rpi-imager
```

### Write the Image

```bash
rpi-imager
```

1. Click **Choose Device** → select your Pi model (`cp-01` = Raspberry Pi 5, `wrk-01` = Raspberry Pi 4)
2. Click **Choose OS** → `Raspberry Pi OS (Other)` → `Raspberry Pi OS Lite (64-bit)`
3. Click **Choose Storage** → select your microSD card or USB SSD
4. Click **Next**
5. When asked about customisation → click **Edit Settings**

### Customisation Settings

**General tab:**

| Field | cp-01 | wrk-01 |
|-------|-------|--------|
| Hostname | `cp-01` | `wrk-01` |
| Username | `kubeadmin` | `kubeadmin` |
| Password | `kubeadmin` | `kubeadmin` |
| Timezone | `Australia/Brisbane` | `Australia/Brisbane` |
| Keyboard | `us` | `us` |

**Services tab:**
- Enable SSH: tick
- Select **Allow public-key authentication only**
- Paste your public key contents from `~/.ssh/pi_key.pub`

**WiFi:** leave blank — ethernet only.

Click **Save** → **Yes** to apply → confirm the write and wait for it to finish.

---

## 3.5 UniFi Switch Port Configuration

Before booting the Pis, ensure the switch ports are assigned to the correct VLAN.

In UniFi → Devices → USW Flex Mini → Port Manager:

| Port | Device | Native VLAN |
|------|--------|-------------|
| 4 | cp-01 | Home LAB (30) |
| 5 | wrk-01 | Home LAB (30) |

> **Important:** If ports are left on Default (VLAN 1), systemd-networkd will stall on boot waiting for a gateway response on `192.168.30.x` that never comes, causing a solid green LED hang.

---

## 3.6 First Boot

1. Safely eject the media from your PC
2. Insert into the Pi and connect the Ethernet cable
3. Power on
4. Wait 2–3 minutes for first boot to complete
5. Find the DHCP address assigned by the router — check your UniFi client list or scan:

```bash
nmap -sn 192.168.30.0/24
nmap -sn 192.168.0.0/24
```

6. SSH in using the DHCP address:

```bash
ssh -i ~/.ssh/pi_key kubeadmin@<dhcp-ip>
```

---

## 3.7 Configure Static IP

Run the following on each Pi. Adjust the `Address` field for `cp-01` vs `wrk-01`.

### cp-01

```bash
sudo mkdir -p /etc/systemd/network

sudo tee /etc/systemd/network/10-eth0-static.network << 'EOF'
[Match]
Name=eth0

[Network]
DHCP=no
LinkLocalAddressing=no
Address=192.168.30.20/24
Gateway=192.168.30.1
DNS=1.1.1.1
DNS=192.168.30.1
EOF
```

### wrk-01

```bash
sudo mkdir -p /etc/systemd/network

sudo tee /etc/systemd/network/10-eth0-static.network << 'EOF'
[Match]
Name=eth0

[Network]
DHCP=no
LinkLocalAddressing=no
Address=192.168.30.21/24
Gateway=192.168.30.1
DNS=1.1.1.1
DNS=192.168.30.1
EOF
```

### Enable systemd-networkd

```bash
sudo systemctl enable systemd-networkd --now
```

Your SSH session will drop when the IP changes. Reconnect using the static IP:

```bash
# cp-01
ssh cp-01

# wrk-01
ssh wrk-01
```

### Disable NetworkManager

NetworkManager must be removed to prevent it from reassigning DHCP addresses and conflicting with pod networking:

```bash
sudo systemctl stop NetworkManager

sudo systemctl disable NetworkManager
```

### Fix DNS

The `resolv.conf` symlink points to a file that does not exist without systemd-resolved. Replace it with a static file:

```bash
sudo rm -f /etc/resolv.conf

sudo bash -c 'echo "nameserver 1.1.1.1" > /etc/resolv.conf'

sudo bash -c 'echo "nameserver 192.168.30.1" >> /etc/resolv.conf'
```

Verify DNS is working:

```bash
ping -c 2 deb.debian.org
```

---

## 3.8 Disable Swap

Kubernetes requires swap to be completely disabled on all nodes:

```bash
sudo swapoff -a

sudo sed -i '/swap/s/^/#/' /etc/fstab

sudo rm -f /swap.img
```

If zram swap is present (Pi 4):

```bash
sudo swapoff /dev/zram0

sudo systemctl disable --now zramswap 2>/dev/null || true
```

Verify swap is off:

```bash
free -h | grep Swap
```

Expected output:

```
Swap:          0B          0B          0B
```

---

## 3.9 Install Required Packages

```bash
sudo apt update

sudo apt install -y \
    curl wget git net-tools htop vim \
    apt-transport-https ca-certificates \
    gnupg lsb-release containerd
```

---

## 3.10 Configure containerd

K3s uses containerd as its container runtime. It must be configured to use the systemd cgroup driver to match Kubernetes expectations:

```bash
sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd

sudo systemctl enable containerd
```

Verify containerd is running:

```bash
sudo systemctl status containerd --no-pager | head -3

sudo ctr version
```

---

## 3.11 Load Kernel Modules

Kubernetes requires two kernel modules to be loaded — `overlay` for container filesystem layering, and `br_netfilter` for pod network traffic routing:

```bash
sudo modprobe overlay

sudo modprobe br_netfilter
```

Make them load automatically on reboot:

```bash
sudo tee /etc/modules-load.d/k8s.conf << 'EOF'
overlay
br_netfilter
EOF
```

Configure the required sysctl network parameters:

```bash
sudo tee /etc/sysctl.d/k8s.conf << 'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Apply immediately and verify:

```bash
sudo sysctl --system | grep -E "bridge-nf|ip_forward"
```

---

## 3.12 Enable cgroup Boot Flags

The Raspberry Pi kernel does not enable memory cgroups by default. The following appends the required flags to `cmdline.txt`. The script checks first to avoid duplicating flags on repeated runs:

```bash
CMDLINE_FILE="/boot/firmware/cmdline.txt"

CGROUP_FLAGS="cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1 swapaccount=1 systemd.unified_cgroup_hierarchy=1"

CURRENT_CMDLINE=$(cat "$CMDLINE_FILE")
if [[ "$CURRENT_CMDLINE" != *"cgroup_enable=memory"* ]]; then
    sudo sed -i "1s/$/ $CGROUP_FLAGS/" "$CMDLINE_FILE"
fi

cat /boot/firmware/cmdline.txt
```

Verify the flags appear in the output, then reboot to apply:

```bash
sudo reboot
```

---

## 3.13 Verify Node is Ready

After reboot, SSH back in and run the following checks:

```bash
hostname

ip a show eth0

sudo systemctl status containerd --no-pager | head -3

cat /proc/cmdline | grep -o "cgroup_enable=memory"

free -h | grep Swap

ping -c 2 deb.debian.org
```

Expected results:

| Check | cp-01 | wrk-01 |
|-------|-------|--------|
| Hostname | `cp-01` | `wrk-01` |
| IP | `192.168.30.20` | `192.168.30.21` |
| containerd | `active (running)` | `active (running)` |
| cgroup | `cgroup_enable=memory` | `cgroup_enable=memory` |
| Swap | `0B 0B 0B` | `0B 0B 0B` |
| DNS | ping succeeds | ping succeeds |

---

## 3.14 Verification Checklist

Complete this checklist for each node before proceeding to Document 4.

### Per Node

- [ ] Flashed with Raspberry Pi OS Lite (64-bit) Bookworm
- [ ] Imager customisation applied — hostname, kubeadmin user, SSH key, timezone
- [ ] UniFi switch port set to Home LAB (VLAN 30)
- [ ] Static IP configured via systemd-networkd
- [ ] NetworkManager disabled
- [ ] DNS `resolv.conf` set to `1.1.1.1` and `192.168.30.1`
- [ ] Swap disabled — confirmed with `free -h`
- [ ] Required packages installed including containerd
- [ ] containerd configured with `SystemdCgroup = true`
- [ ] Kernel modules loaded — `overlay` and `br_netfilter`
- [ ] sysctl network parameters configured
- [ ] cgroup flags added to `cmdline.txt`
- [ ] Node verified after reboot — all checks pass

### Boot and Install Order

- [ ] `cp-01` fully prepared and verified
- [ ] `wrk-01` fully prepared and verified
- [ ] Both nodes confirmed ready before proceeding to K3s installation

---

*Next: Document 4 — Installing K3s*
