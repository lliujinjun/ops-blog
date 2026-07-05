---
title: "☸️ Single-Node Kubernetes with CRI-O and Calico on CentOS Stream 9"
date: 2026-07-05T13:58:00+08:00
draft: false
description: "Step-by-step guide to installing a single-node Kubernetes cluster using kubeadm, CRI-O runtime, and Calico CNI on CentOS Stream 9."
tags: ["kubernetes", "kubeadm", "cri-o", "calico", "centos"]
categories: ["containers"]
---

A lightweight, single-node Kubernetes cluster using CRI-O as the container runtime and Calico for networking — all on CentOS Stream 9.

<!--more-->

---

## Why CRI-O Instead of containerd?

CRI-O is a lightweight container runtime built specifically for Kubernetes. It implements the CRI (Container Runtime Interface) directly — no Docker shim, no extra layers. It's what OpenShift uses, it's what many production clusters run, and it integrates tightly with kubelet.

The tradeoff: CRI-O is Kubernetes-only. You can't run `docker run` with it. But for a Kubernetes node, that's exactly what you want.

---

## Prerequisites

- CentOS Stream 9 VM
- 3.5GB+ RAM
- Root/sudo access
- Hostname set (we'll use `k8s-node1`)

---

## Step 1: System Prep

Disable swap, set SELinux to permissive, and configure kernel parameters:

```bash
sudo hostnamectl set-hostname k8s-node1
echo '192.168.211.134 k8s-node1' | sudo tee -a /etc/hosts

sudo swapoff -a
sudo sed -i "/swap/d" /etc/fstab

sudo setenforce 0
sudo sed -i "s/^SELINUX=enforcing/SELINUX=permissive/" /etc/selinux/config

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<"EOF" | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

Open firewall ports:

```bash
sudo firewall-cmd --permanent --add-port={6443,2379-2380,10250,10257,10259}/tcp
sudo firewall-cmd --permanent --add-port=4789/udp
sudo firewall-cmd --reload
```

---

## Step 2: Install CRI-O

Add the CRI-O repository from `pkgs.k8s.io` and install:

```bash
cat <<"EOF" | sudo tee /etc/yum.repos.d/cri-o.repo
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

Verify:

```bash
sudo crio --version
# crio version 1.31.5
sudo systemctl is-active crio
# active
```

### Configure CRI-O for Your Region

If you're in China, `registry.k8s.io` may be slow or unreachable. Configure CRI-O to use a local mirror for the pause image:

```bash
echo 'pause_image = "registry.aliyuncs.com/google_containers/pause:3.10"' \
  | sudo tee -a /etc/crio/crio.conf.d/10-crio.conf

sudo systemctl restart crio
```

---

## Step 3: Install kubeadm, kubelet, kubectl

```bash
cat <<"EOF" | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
EOF

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

---

## Step 4: Initialize the Cluster

```bash
sudo kubeadm init \
  --cri-socket unix:///var/run/crio/crio.sock \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=192.168.211.134 \
  --image-repository=registry.aliyuncs.com/google_containers
```

> The `--image-repository` flag is essential if `registry.k8s.io` is blocked in your region. If you have direct access, you can omit it.

When it finishes, you'll see:

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, run:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Step 5: Install Calico CNI

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/custom-resources.yaml
```

Wait a minute for Calico to deploy:

```bash
kubectl get nodes
```

The node should transition from `NotReady` to `Ready` once Calico networking is established.

---

## Step 6: Verify

```bash
kubectl get nodes -o wide
```

```
NAME        STATUS   ROLES           AGE   VERSION    INTERNAL-IP       OS-IMAGE          CONTAINER-RUNTIME
k8s-node1   Ready    control-plane   93s   v1.31.14   192.168.211.134   CentOS Stream 9   cri-o://1.31.5
```

```bash
kubectl get pods -A
```

All pods should be `Running` or `Completed`:

```
NAMESPACE          NAME                                       READY   STATUS    RESTARTS
calico-system      calico-node-xxxxx                          1/1     Running   0
calico-system      calico-typha-xxxxx                         1/1     Running   0
kube-system        coredns-xxxxx                              1/1     Running   0
kube-system        etcd-k8s-node1                             1/1     Running   0
kube-system        kube-apiserver-k8s-node1                   1/1     Running   0
kube-system        kube-controller-manager-k8s-node1          1/1     Running   0
kube-system        kube-proxy-xxxxx                           1/1     Running   0
kube-system        kube-scheduler-k8s-node1                   1/1     Running   0
```

---

## Deploy a Test App

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort

kubectl get svc

# Access it
curl http://localhost:$(kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')
```

---

## Access from Outside

On the VM:

```bash
kubectl cluster-info
```

To use `kubectl` from your local machine, copy the kubeconfig:

```bash
scp user@k8s-node1:~/.kube/config ~/.kube/config
# Edit the server address to point to the VM's IP
kubectl get nodes
```

---

## Troubleshooting

### Image pull failures / EOF errors

If you see errors like:

```
parsing image configuration: Get "https://d39mqg4b1dx9z1.cloudfront.net/...": EOF
```

This is `registry.k8s.io` routing through CloudFront, which is often throttled or blocked in certain regions. Fixes:

1. Use `--image-repository=registry.aliyuncs.com/google_containers` with `kubeadm init`
2. Configure CRI-O's pause image to the same mirror (see Step 2)
3. Or use a local registry/proxy

### CRI-O won't start after config change

Duplicate `[crio.image]` sections in TOML config will cause a fatal error. Check the config file and ensure sections aren't duplicated.

### Node stuck in NotReady

Check Calico pods:

```bash
kubectl get pods -n calico-system
kubectl logs -n calico-system calico-node-xxxxx
```

---

## Cleanup

```bash
sudo kubeadm reset -f --cri-socket unix:///var/run/crio/crio.sock
sudo dnf remove -y kubelet kubeadm kubectl cri-o
sudo rm -rf /etc/kubernetes /var/lib/kubelet /etc/cni /etc/crio
```

---

## Key Takeaways

- **CRI-O is purpose-built for Kubernetes** — lighter than containerd+Docker for cluster nodes
- **kubeadm** makes cluster bootstrap simple — one command and you're running
- **Calico** provides robust pod networking with NetworkPolicy support
- **Image mirrors** are essential in restricted network environments
- **Single-node clusters** are perfect for development, testing, and learning Kubernetes internals

---

*Built on CentOS Stream 9 with Kubernetes 1.31, CRI-O 1.31.5, and Calico 3.29. Not for production use.*
