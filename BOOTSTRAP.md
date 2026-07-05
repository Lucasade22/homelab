# Yggdrasil Homelab Bootstrap Guide

This document describes the steps required to bootstrap the cluster from scratch.
Flux manages all workloads declaratively via this repo, but some bootstrap steps
must be performed manually before Flux can take over.

## Prerequisites
- Ubuntu Server installed on the node
- SSH key authentication configured
- Tailscale installed and connected

## Step 1 — Install K3s

K3s must be installed with Flannel disabled and kube-proxy disabled so Cilium
can manage all networking:

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --write-kubeconfig-mode 644 \
  --tls-san 192.168.0.120 \
  --tls-san 100.75.30.28 \
  --disable traefik \
  --disable servicelb \
  --flannel-backend=none \
  --disable-network-policy \
  --disable-kube-proxy
```

## Step 2 — Install Cilium (bootstrap only)

Cilium must be installed directly via Helm before Flux can run, because Flux
itself needs pod networking to reach the Kubernetes API. After this bootstrap
install, Flux takes over management of Cilium via the HelmRelease in git.

```bash
helm repo add cilium https://helm.cilium.io
helm repo update
helm install cilium cilium/cilium \
  --version 1.17.3 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<NODE_LAN_IP> \
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

Wait for Cilium and CoreDNS to be Ready before proceeding.

## Step 3 — Bootstrap Flux

```bash
flux bootstrap github \
  --owner=Lucasade22 \
  --repository=homelab \
  --branch=main \
  --path=clusters/yggdrasil \
  --personal
```

Flux will take over from here, reconciling all infrastructure and applications
from this repository.

## Step 4 — UFW Rules

Apply UFW rules after Flux has reconciled (see infrastructure/host/ufw-rules.md)
