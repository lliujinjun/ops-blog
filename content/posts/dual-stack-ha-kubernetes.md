---
title: "☸️ Dual-Stack HA Kubernetes Cluster on CentOS Stream 9"
date: 2026-07-05T15:48:00+08:00
draft: false
description: "Building a highly available, dual-stack (IPv4 + IPv6) Kubernetes cluster across 3 CentOS Stream 9 VMs using Keepalived, HAProxy, kubeadm, CRI-O, and Calico."
tags: ["kubernetes", "dual-stack", "ha", "keepalived", "haproxy", "cri-o", "calico", "centos"]
categories: ["kubernetes", "infrastructure"]
---

A production-grade, dual-stack Kubernetes cluster is a mouthful. Let's build one step by step.

<!--more-->

---

## What We're Building

A 3-node highly available Kubernetes cluster with both IPv4 and IPv6 networking, fronted by a floating VIP and load-balanced through HAProxy.

```
        192.168.211.100:16443  ← VIP (Keepalived)
               │
          ┌────┴────┐
          │ HAProxy │  (round-robin to all 3 API servers)
          └────┬────┘
    ┌──────────┼──────────┐
    │          │          │
┌───┴────┐ ┌───┴────┐ ┌───┴────┐
│ k8s-m1 │ │ k8s-m2 │ │ k8s-m3 │
│:6443   │ │:6443   │ │:6443   │
│192.168 │ │192.168 │ │192.168 │
│.211.134│ │.211.135│ │.211.136│
│fd00    │ │fd00    │ │fd00    │
│::134   │ │::135   │ │::136   │
└────────┘ └────────┘ └────────┘
   etcd      etcd      etcd     ← stacked
```

### Dual-Stack Layout

| Layer | IPv4 | IPv6 |
|---|---|---|
| **Node network** | 192.168.211.0/24 | fd00::/64 (ULA) |
| **Pod network** | 192.168.0.0/16 | fd00:10:244::/56 |
| **Service network** | 10.96.0.0/12 | fd00:10:96::/112 |

Every pod gets both an IPv4 and an IPv6 address. Services work on both protocols. Everything is redundant — any single node can fail and the cluster keeps running.

---

## Architecture

### Keepalived

Runs on all 3 nodes. Elects a MASTER (priority 150 → 120 → 110). Whoever wins holds the VIP `192.168.211.100`. If the MASTER's HAProxy dies, priority drops by 50, and the VIP moves to the next node.

### HAProxy

Runs on all 3 nodes, listens on `:16443`. Load balances across all 3 API servers (`:6443`). Health checks every 2 seconds — if an API server goes down, HAProxy stops routing to it.

### kubeadm

Bootstraps the first control plane node with `--control-plane-endpoint=192.168.211.100:16443`. The other two nodes join as control plane replicas. etcd runs stacked (on each control plane node) with 3-member quorum.

### Calico

Provides pod networking. Configured with dual-stack IP pools — one IPv4 pool and one IPv6 pool, matching the pod CIDRs passed to kubeadm.

### CRI-O

Lightweight container runtime built specifically for Kubernetes. No Docker shim, no extra layers.

---

## Prerequisites

- 3 CentOS Stream 9 VMs with 3.5GB+ RAM each
- Root/sudo access on all
- All VMs on the same subnet
- IPv6 enabled (link-local is fine, we'll add global addresses)

Our node spec:

| VM | Hostname | IPv4 | IPv6 (ULA) |
|---|---|---|---|
| remote-vm1 | **k8s-m1** | 192.168.211.134 | fd00::134 |
| remote-vm2 | **k8s-m2** | 192.168.211.135 | fd00::135 |
| remote-vm3 | **k8s-m3** | 192.168.211.136 | fd00::136 |
| **VIP** | **k8s-vip** | 192.168.211.100 | — |

---

## Phase 1: System Prerequisites

Run on **all 3 nodes**:

```bash
# Hostnames (run with appropriate name per node)
sudo hostnamectl set-hostname k8s-m1

# Add IPv6 ULA address
sudo nmcli con mod ens33 ipv6.addresses fd00::134/64
sudo nmcli con mod ens33 ipv6.method manual
sudo nmcli con up ens33

# Hosts entries
echo -e '192.168.211.134 k8s-m1\nfd00::134 k8s-m1' | sudo tee -a /etc/hosts
echo -e '192.168.211.135 k8s-m2\nfd00::135 k8s-m2' | sudo tee -a /etc/hosts
echo -e '192.168.211.136 k8s-m3\nfd00::136 k8s-m3' | sudo tee -a /etc/hosts
echo -e '192.168.211.100 k8s-vip\nfd00::100 k8s-vip' | sudo tee -a /etc/hosts

# Disable swap
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# SELinux permissive
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config

# Kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl for dual-stack
cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
sudo sysctl --system

# Firewall - open Kubernetes + HAProxy ports
sudo firewall-cmd --permanent --add-port={6443,2379-2380,10250,10257,10259,16443}/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --permanent --add-protocol=vrrp
sudo firewall-cmd --reload
```

> **Important:** Don't forget port `16443` — that's the HAProxy frontend. This is subtle and easy to miss.

---

## Phase 2: Install CRI-O

```bash
# Add CRI-O repo
cat <<'EOF' | sudo tee /etc/yum.repos.d/cri-o.repo
[cri-o]
name=cri-o
baseurl=https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/addons:/cri-o:/stable:/v1.31/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y cri-o
sudo systemctl enable --now crio
```

### Configure CRI-O for Regional Mirrors

If you're in a region where `registry.k8s.io` is slow or blocked (China, for example), configure CRI-O to use a mirror:

```bash
# Create registry mirror config
cat <<'EOF' | sudo tee /etc/containers/registries.conf.d/mirror.conf
[[registry]]
location = "registry.k8s.io"
[[registry.mirror]]
location = "registry.aliyuncs.com/google_containers"
EOF

# Set pause image to match
echo 'pause_image = "registry.aliyuncs.com/google_containers/pause:3.10"' \
  | sudo tee /etc/crio/crio.conf.d/10-crio.conf

sudo systemctl restart crio
```

---

## Phase 3: Install Keepalived + HAProxy

```bash
sudo dnf install -y keepalived haproxy
```

### HAProxy Config

Same on all 3 nodes (`/etc/haproxy/haproxy.cfg`):

```conf
global
    log /dev/log local0
    maxconn 4096
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    retries 3
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend k8s_frontend
    bind *:16443
    mode tcp
    default_backend k8s_api_servers

backend k8s_api_servers
    mode tcp
    balance roundrobin
    option tcp-check
    server k8s-m1 192.168.211.134:6443 check fall 3 rise 2
    server k8s-m2 192.168.211.135:6443 check fall 3 rise 2
    server k8s-m3 192.168.211.136:6443 check fall 3 rise 2
```

### Keepalived Config

**k8s-m1 (MASTER, priority 150)** — `/etc/keepalived/keepalived.conf`:

```conf
global_defs {
    router_id k8s-m1
    enable_script_security
}

vrrp_script check_haproxy {
    script "/usr/bin/pgrep haproxy"
    interval 2
    weight -50
    fall 2
    rise 2
    user root
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k8sha
    }
    virtual_ipaddress {
        192.168.211.100/24
    }
    track_script {
        check_haproxy
    }
}
```

**k8s-m2 (BACKUP, priority 120):** Same config but `router_id k8s-m2`, `state BACKUP`, `priority 120`.
**k8s-m3 (BACKUP, priority 110):** Same config but `router_id k8s-m3`, `state BACKUP`, `priority 110`.

Start everything:

```bash
sudo systemctl enable --now haproxy
sudo systemctl enable --now keepalived
```

---

## Phase 4: Install kubeadm Tools

```bash
cat <<'EOF' | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubelet kubeadm kubectl conntrack-tools --disableexcludes=kubernetes
sudo systemctl enable kubelet
```

---

## Phase 5: Bootstrap the Cluster

### Initialize k8s-m1

```bash
# If using a mirror registry in restricted regions:
sudo kubeadm init \
  --cri-socket unix:///var/run/crio/crio.sock \
  --image-repository=registry.aliyuncs.com/google_containers \
  --control-plane-endpoint=192.168.211.100:16443 \
  --apiserver-advertise-address=192.168.211.134 \
  --pod-network-cidr=192.168.0.0/16,fd00:10:244::/56 \
  --service-cidr=10.96.0.0/12,fd00:10:96::/112 \
  --kubernetes-version=v1.31.14 \
  --node-name=k8s-m1 \
  --upload-certs
```

The `--control-plane-endpoint` points to the VIP + HAProxy port. The `--pod-network-cidr` and `--service-cidr` take two comma-separated values for dual-stack (IPv4 first, IPv6 second).

When it finishes, it outputs a join command. Save it:

```
kubeadm join 192.168.211.100:16443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <key>
```

### Set up kubectl

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Join k8s-m2 and k8s-m3

```bash
# On k8s-m2
sudo kubeadm join 192.168.211.100:16443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <key> \
  --node-name=k8s-m2 \
  --cri-socket unix:///var/run/crio/crio.sock

# On k8s-m3 (same token, same key)
sudo kubeadm join 192.168.211.100:16443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <key> \
  --node-name=k8s-m3 \
  --cri-socket unix:///var/run/crio/crio.sock
```

---

## Phase 6: Install Calico with Dual-Stack

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml

cat <<'EOF' | kubectl apply -f -
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
    - blockSize: 122
      cidr: fd00:10:244::/56
      encapsulation: None
      natOutgoing: Enabled
      nodeSelector: all()
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

Wait a minute for Calico to deploy. If nodes stay `NotReady` for too long, check if Calico images are being pulled — in restricted networks you may need to pre-pull them:

```bash
for img in node cni kube-controllers typha pod2daemon-flexvol; do
  sudo crictl pull docker.io/calico/$img:v3.29.2
done
```

### Remove taints for scheduling

Since we only have control plane nodes (no worker nodes), remove the taint to allow normal pods to run:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## Phase 7: Verify

```bash
kubectl get nodes -o wide
```

```
NAME     STATUS   ROLES           AGE   VERSION    INTERNAL-IP       OS-IMAGE          CONTAINER-RUNTIME
k8s-m1   Ready    control-plane   14m   v1.31.14   192.168.211.134   CentOS Stream 9   cri-o://1.31.5
k8s-m2   Ready    control-plane   6m    v1.31.14   192.168.211.135   CentOS Stream 9   cri-o://1.31.5
k8s-m3   Ready    control-plane   5m    v1.31.14   192.168.211.136   CentOS Stream 9   cri-o://1.31.5
```

### Verify Dual-Stack

```bash
# Deploy a test pod
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- /bin/sh -c "ip addr"

# You should see both eth0@if... inet (IPv4) and inet6 (IPv6) addresses
```

### Verify HA

```bash
# Test the VIP endpoint
curl -sk https://192.168.211.100:16443/healthz
# → "ok"

# Check etcd cluster health
kubectl get pods -n kube-system | grep etcd
# All 3 etcd pods should be Running
```

### Test Failover

```bash
# On the node holding the VIP, stop HAProxy
sudo systemctl stop haproxy

# The VIP should move to another node within ~5 seconds
# Check from another node:
ip addr show ens33 | grep 192.168.211.100

# The API should still be accessible via the VIP
curl -sk https://192.168.211.100:16443/healthz
# → "ok"

# Restore HAProxy on the original node
sudo systemctl start haproxy
```

---

## Troubleshooting

### Port 16443 not reachable from other nodes

Add the HAProxy port to the firewall on all nodes:

```bash
sudo firewall-cmd --permanent --add-port=16443/tcp
sudo firewall-cmd --reload
```

### kubeadm join stuck at preflight checks

The join command needs to pull container images. If `registry.k8s.io` is blocked, pre-pull images from a mirror:

```bash
for img in kube-apiserver:v1.31.14 kube-controller-manager:v1.31.14 \
  kube-scheduler:v1.31.14 kube-proxy:v1.31.14 coredns:v1.11.3 \
  etcd:3.5.24-0 pause:3.10; do
  sudo crictl pull registry.aliyuncs.com/google_containers/$img
done
```

### "No route to host" when connecting to VIP

Check reverse path filtering:

```bash
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.ens33.rp_filter=0
echo 'net.ipv4.conf.all.rp_filter = 0' | sudo tee -a /etc/sysctl.d/k8s.conf
```

### Calico pods stuck in Init

Pull Calico images manually:

```bash
for img in node cni kube-controllers typha pod2daemon-flexvol; do
  sudo crictl pull docker.io/calico/$img:v3.29.2
done
```

---

## Summary

| Component | What We Used | Why |
|---|---|---|
| **Load balancer** | Keepalived + HAProxy | Simple, battle-tested, handles failover + load balancing |
| **Container runtime** | CRI-O | Purpose-built for Kubernetes, lightweight, no Docker dependency |
| **Cluster bootstrap** | kubeadm | The standard tool, handles certs, tokens, and control plane joins |
| **CNI** | Calico | Dual-stack support, NetworkPolicy, VXLAN encapsulation |
| **Networking** | Dual-stack (IPv4 + IPv6) | Future-proof, both protocols work simultaneously |

---

## Key Takeaways

- **3 control plane nodes + stacked etcd** — survive any single node failure
- **Dual-stack** — pods get both IPv4 and IPv6, services work on both protocols
- **Keepalived + HAProxy** is the simplest HA pattern that actually works
- **CRI-O** is lighter than containerd for Kubernetes-only nodes
- **Image mirrors** are essential in restricted networks — configure them before running kubeadm
- **Firewall ports** are the most common gotcha — double-check every port

---

*Built on CentOS Stream 9 with Kubernetes 1.31, CRI-O 1.31.5, Calico 3.29, Keepalived 2.2.8, and HAProxy 2.8.*
