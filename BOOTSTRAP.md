# Yggdrasil Homelab Bootstrap Guide

This document covers the complete process of bootstrapping a new node from
a fresh Ubuntu Server installation to a fully running cluster managed by Flux.

---

## Phase 1 — Ubuntu Server Initial Setup

### 1.1 — Create a non-root user with sudo (if not done during install)

```bash
adduser lucas
usermod -aG sudo lucas
su - lucas
```

### 1.2 — Set a static hostname

```bash
sudo hostnamectl set-hostname asgard
```

Update `/etc/hosts` to resolve the new hostname:

```bash
sudo nano /etc/hosts
# Add or update this line:
127.0.1.1    asgard
```

### 1.3 — Configure SSH key authentication

On your Mac, copy your public key to the server:

```bash
ssh-copy-id lucas@<SERVER_LAN_IP>
```

Then on the server, disable password and root SSH login:

```bash
sudo nano /etc/ssh/sshd_config
# Set these values:
PasswordAuthentication no
PermitRootLogin no

sudo systemctl restart ssh
```

### 1.4 — Reserve the server IP on your router

Log into your router admin panel and create a DHCP reservation for the
server's MAC address. Assign it a static IP from the server range:
192.168.0.120  — asgard (control plane)
192.168.0.121  — future second node

---

## Phase 2 — Tailscale

### 2.1 — Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the authentication URL printed to your terminal.

### 2.2 — Set the Tailscale hostname

```bash
sudo tailscale set --hostname=asgard
```

### 2.3 — Note the Tailscale IP

```bash
tailscale ip -4
# Should return 100.75.30.28 for asgard
```

---

## Phase 3 — UFW Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw allow from 192.168.0.0/24 to any port 22 proto tcp comment 'SSH from LAN fallback'
sudo ufw logging on
sudo ufw enable
```

Verify with `sudo ufw status verbose` before proceeding.

Additional rules added after cluster is running:

```bash
sudo ufw allow from 192.168.0.0/24 to any port 6443 proto tcp comment 'k3s API from LAN'
sudo ufw allow from 192.168.0.0/24 to 192.168.0.120 port 80 proto tcp comment 'HTTP LAN to internal ingress'
sudo ufw allow from 192.168.0.0/24 to 192.168.0.120 port 443 proto tcp comment 'HTTPS LAN to internal ingress'
sudo ufw deny from 192.168.0.0/24 to 100.75.30.28 port 80 proto tcp comment 'Block LAN to external ingress HTTP'
sudo ufw deny from 192.168.0.0/24 to 100.75.30.28 port 443 proto tcp comment 'Block LAN to external ingress HTTPS'
sudo ufw allow from 192.168.0.0/24 to 192.168.0.120 port 8123 proto tcp comment 'Home Assistant from LAN devices'
```

---

## Phase 4 — K3s Installation

K3s must be installed with Flannel and kube-proxy disabled so Cilium
can manage all networking:

```bash
curl -sfL https://get.k3s.io | sh -s - server --write-kubeconfig-mode 644 --tls-san 192.168.0.120 --tls-san 100.75.30.28 --disable traefik --disable servicelb --flannel-backend=none --disable-network-policy --disable-kube-proxy
```

Verify K3s started — node will show NotReady until Cilium is installed:

```bash
sudo systemctl status k3s
kubectl get nodes
```

---

## Phase 5 — Cilium Bootstrap Install

⚠️ This step must be performed from your Mac, not the server.
Cilium must be installed manually before Flux can run because Flux itself
needs pod networking to reach the Kubernetes API. After bootstrap,
Flux takes over Cilium management via the HelmRelease in git.

### 5.1 — Set up kubeconfig on your Mac

On the server, print the kubeconfig:

```bash
sudo cat /etc/rancher/k3s/k3s.yaml
```

On your Mac, create the kubeconfig:

```bash
mkdir -p ~/.kube
nano ~/.kube/config
# Paste the contents, changing server: https://127.0.0.1:6443
# to server: https://100.75.30.28:6443
chmod 600 ~/.kube/config
```

Verify connectivity:

```bash
kubectl get nodes
# Should show asgard as NotReady
```

### 5.2 — Install Cilium via Helm

```bash
helm repo add cilium https://helm.cilium.io
helm repo update

helm install cilium cilium/cilium \
  --version 1.17.3 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.0.120 \
  --set k8sServicePort=6443 \
  --set ipam.mode=kubernetes \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set operator.replicas=1 \
  --set routingMode=tunnel \
  --set tunnelProtocol=vxlan \
  --set bpf.masquerade=true \
  --set socketLB.enabled=true \
  --set cgroup.autoMount.enabled=false \
  --set cgroup.hostRoot=/sys/fs/cgroup \
  --set endpointRoutes.enabled=true
```

### 5.3 — Wait for Cilium and CoreDNS to be Ready

```bash
kubectl get pods -n kube-system --watch
```

Wait until these are all 1/1 Running before proceeding:
- cilium-*
- cilium-operator-*
- coredns-*

Node should now show Ready:

```bash
kubectl get nodes
# asgard   Ready   control-plane
```

---

## Phase 6 — Flux Bootstrap

⚠️ Run from your Mac.

### 6.1 — Install Flux CLI

```bash
brew install fluxcd/tap/flux
```

### 6.2 — Create a GitHub Personal Access Token

Go to GitHub → Settings → Developer settings → Personal access tokens →
Tokens (classic) → Generate new token (classic)

- Name: flux-homelab
- Scopes: repo (top level)

### 6.3 — Bootstrap Flux

```bash
export GITHUB_TOKEN=<your-token>
flux bootstrap github \
  --owner=Lucasade22 \
  --repository=homelab \
  --branch=main \
  --path=clusters/yggdrasil \
  --personal
```

Flux will install itself into the cluster and commit its configuration
to the repo. From this point, the repo is the source of truth for the
entire cluster.

### 6.4 — Verify Flux is reconciling

```bash
flux get kustomizations --watch
```

All kustomizations should reach Ready: True within a few minutes.

---

## Phase 7 — Windows kubectl Setup

Install kubectl:

```powershell
winget install -e --id Kubernetes.kubectl
```

Create kubeconfig (same content as Mac, same server URL change):

```powershell
mkdir $HOME\.kube -Force
notepad $HOME\.kube\config
# Save as type: All Files, filename: config (no extension)
```

---

## Phase 8 — Post-Bootstrap Verification

```bash
# All pods running
kubectl get pods -A

# All Helm releases healthy
flux get helmreleases -A

# Services responding
curl -I https://homepage.bifrost22.dev
curl -I https://grafana.bifrost22.dev
curl -I https://homeassistant.bifrost22.dev
```

---

## Adding a Second Node (Future)

When adding midgard or another node:

1. Repeat Phases 1-3 on the new machine
2. On asgard, get the node join token:
```bash
   sudo cat /var/lib/rancher/k3s/server/node-token
```
3. On the new node, join the cluster:
```bash
   curl -sfL https://get.k3s.io | K3S_URL=https://192.168.0.120:6443 K3S_TOKEN=<token> sh -
```
4. Deploy Longhorn for distributed storage
5. Configure Keepalived for DNS HA

---

## IP Scheme Reference
192.168.0.1        — Router/gateway
192.168.0.120      — asgard (control plane)
192.168.0.121      — future: midgard (worker node)
192.168.0.150-199  — DHCP pool for regular devices
192.168.0.220      — Keepalived VIP (DNS, future)
192.168.0.221+     — Additional VIPs as needed
Tailscale:
100.75.30.28       — asgard