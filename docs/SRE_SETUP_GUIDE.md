# Document 1 — SRE Setup Guide

**Phase 4 — Kubernetes on Raspberry Pi**  
**Version:** 1.0  
**Management Machine:** `sre-mgmnt-01`

---

## Overview

This document covers the setup of `sre-mgmnt-01` — your management machine. It is an Arch Linux VM that runs `kubectl`, `k9s`, and connects to the Raspberry Pi cluster remotely. Kubernetes does not run on this machine. All cluster management commands in Phase 4 are issued from here.

---

## 1.1 Update Arch Linux

SSH into the management VM:

```bash
ssh sre-mgmnt-01
```

Run a full system update:

```bash
sudo pacman -Syu --noconfirm
```

Expected output:

```
:: Synchronizing package databases...
:: Starting full system upgrade...
...
(100%) done.
```

If any updates were applied, reboot before continuing:

```bash
sudo reboot now
```

---

## 1.2 Install Kubernetes Admin Tools

```bash
sudo pacman -S --needed --noconfirm \
kubectl \
k9s \
git \
curl \
wget
```

What each package does:

| Package | Purpose |
| :-------- | :------- |
| `kubectl` | Kubernetes command-line tool — your primary interface to the cluster. Used to deploy applications, inspect resources, view logs, and troubleshoot. |
| `k9s` | Terminal dashboard for browsing cluster resources interactively. Simplifies cluster management — monitor pods, view real-time logs, and exec into containers without long `kubectl` commands. |
| `git` | Version control. Used for cloning repositories, managing infrastructure configuration files, and tracking changes to deployment manifests. |
| `curl` | Transfers data with URLs. Used to fetch Kubernetes manifests, download installation scripts, and test API endpoints running inside the cluster. |
| `wget` | Network file downloader. Used to download packages, configuration templates, and cluster binaries over HTTP, HTTPS, or FTP. |

---

## 1.3 Configure k9s

Create the k9s configuration directory:

```bash
mkdir -p ~/.config/k9s
```

Write the configuration file:

```bash
cat > ~/.config/k9s/config.yaml << 'EOF'
k9s:
  ui:
    currentNamespace: default
    logos: true
    headless: false
  logger:
    tail: 200
    showTime: true
EOF
```

Launch k9s to verify it opens correctly. You will connect it to the cluster in Document 4 — Installing K3s:

```bash
k9s
# Navigate: :pod, :node, :svc, :deploy
# Exit: :q!
```

---

## 1.4 Configure SSH Access to the Raspberry Pi Nodes

### Generate SSH Keys

```bash
ssh-keygen -t ed25519 -C "arch-to-pi" -f ~/.ssh/pi_key
```

Expected output:

```
Generating public/private ed25519 key pair.
Enter passphrase for "/home/sre/.ssh/pi_key" (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/sre/.ssh/pi_key
Your public key has been saved in /home/sre/.ssh/pi_key.pub
The key fingerprint is:
SHA256:obf8Q7E0Bz8Mjjln/Vc+13a1NrzYLOQhu3ZBUMvhgz8 arch-to-pi
```

Verify both key files exist:

```bash
ls -la ~/.ssh/
# -rw------- 1 sre sre 399 pi_key
# -rw-r--r-- 1 sre sre  92 pi_key.pub
```

Copy the public key — you will need it when flashing the Raspberry Pi OS in Document 3:

```bash
cat ~/.ssh/pi_key.pub
```

Copy the full output line. You will paste this into the Raspberry Pi Imager customisation settings.

### Configure SSH Config File

Create or open your SSH config file:

```bash
vim ~/.ssh/config
```

Add the following host blocks:

```bash
Host cp-01
    HostName 192.168.30.20
    User kubeadmin
    IdentityFile ~/.ssh/pi_key

Host wrk-01
    HostName 192.168.30.21
    User kubeadmin
    IdentityFile ~/.ssh/pi_key
```

Save the file. Once configured you can connect to either node instantly:

```bash
ssh cp-01
ssh wrk-01
```

---

## 1.5 Verification Checklist

Before proceeding to Document 2, confirm the following:

- [ ] Arch Linux fully updated and rebooted if required
- [ ] `kubectl`, `k9s`, `git`, `curl`, `wget` installed
- [ ] k9s configuration file created and k9s opens without error
- [ ] SSH key pair generated at `~/.ssh/pi_key` and `~/.ssh/pi_key.pub`
- [ ] Public key copied and ready to paste into Raspberry Pi Imager
- [ ] SSH config file configured for `cp-01` and `wrk-01`

---

*Next: Document 2 — Introduction to Kubernetes*
