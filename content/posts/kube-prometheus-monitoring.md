---
title: "📊 Monitoring Kubernetes Pods with kube-prometheus-stack"
date: 2026-07-05T17:18:00+08:00
draft: false
description: "Install Prometheus, Grafana, and kube-state-metrics on a Kubernetes cluster using Helm — and learn how to monitor pods, nodes, and applications."
tags: ["prometheus", "grafana", "monitoring", "kubernetes", "helm"]
categories: ["kubernetes", "monitoring"]
---

How to set up full-stack monitoring on a Kubernetes cluster — with pre-built dashboards for pods, nodes, and the API server — in under 10 minutes.

<!--more-->

---

## What We're Installing

The **kube-prometheus-stack** Helm chart deploys everything you need:

| Component | What it monitors |
|---|---|
| **Prometheus** | Scrapes, stores, and queries metrics |
| **Grafana** | Dashboards with pre-built panels |
| **Alertmanager** | Routes and sends alerts |
| **node-exporter** | Node-level OS metrics (CPU, RAM, disk, network) |
| **kube-state-metrics** | Kubernetes objects (pods, deployments, nodes) |
| **Prometheus Operator** | Manages Prometheus instances |

Every pod on the cluster becomes visible automatically — CPU, memory, network, restarts, and more — without any extra configuration.

---

## How Pod Monitoring Works

Prometheus monitors pods at three levels:

### Level 1: cAdvisor (built into kubelet)

Every kubelet has cAdvisor embedded. It tracks every container's resource usage and exposes metrics at `/metrics/cadvisor`. Prometheus scrapes this automatically. You immediately get queries like:

```
container_cpu_usage_seconds_total{pod="nginx-xxx"}
container_memory_working_set_bytes{pod="nginx-xxx"}
container_network_receive_bytes_total{pod="nginx-xxx"}
```

### Level 2: kube-state-metrics

This service watches the Kubernetes API and exposes the state of objects:

```
kube_pod_status_phase{phase="Running"}
kube_deployment_status_replicas_available
kube_node_status_condition{condition="Ready"}
```

### Level 3: Application metrics

If your app exposes a `/metrics` endpoint (e.g., a Go app with `promhttp`), Prometheus discovers it through pod annotations or ServiceMonitors:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
```

---

## Prerequisites

- A running Kubernetes cluster (kubeadm or any other)
- Helm installed (if not, install it first)
- kubectl configured

---

## Step 1: Install Helm

If you don't have Helm yet:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
```

---

## Step 2: Install kube-prometheus-stack

```bash
# Add the repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install
helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.service.type=NodePort \
  --set grafana.service.type=NodePort \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

The `--set prometheus.service.type=NodePort` and `--set grafana.service.type=NodePort` expose these services on a node port so you can access them without `kubectl port-forward`.

---

## Step 3: Check the Pods

```bash
kubectl get pods -n monitoring
```

You should see something like:

```
NAME                                                     READY   STATUS
alertmanager-kube-prometheus-kube-prome-alertmanager-0   2/2     Running
kube-prometheus-grafana-64d6f866d7-bxhj9                 3/3     Running
kube-prometheus-kube-prome-operator-6478f697cb-dkqdm     1/1     Running
kube-prometheus-kube-state-metrics-7b5fc8998b-tzkwq      1/1     Running
kube-prometheus-prometheus-node-exporter-cqqtm           1/1     Running
kube-prometheus-prometheus-node-exporter-pq4js           1/1     Running
kube-prometheus-prometheus-node-exporter-sjx88           1/1     Running
prometheus-kube-prometheus-kube-prome-prometheus-0       2/2     Running
```

---

## Step 4: Find Access Points

```bash
kubectl get svc -n monitoring
```

Look for the NodePorts:

```
NAME                                       TYPE        CLUSTER-IP       PORT(S)
kube-prometheus-grafana                    NodePort    10.104.178.186   80:32370/TCP
kube-prometheus-kube-prome-prometheus      NodePort    10.108.212.100   9090:30090/TCP
```

- Grafana: `http://<any-node-ip>:32370`
- Prometheus: `http://<any-node-ip>:30090`

---

## Step 5: Log Into Grafana

Get the admin password:

```bash
kubectl get secret --namespace monitoring \
  -l app.kubernetes.io/component=admin-secret \
  -o jsonpath="{.items[0].data.admin-password}" | base64 --decode
echo
```

Login with `admin` and the password.

---

## Step 6: Explore the Dashboards

Grafana comes with pre-built dashboards:

### Kubernetes / Pods

Shows per-pod metrics:
- CPU usage (cores)
- Memory usage (bytes)
- Network receive/transmit
- Pod restarts
- Container filesystem usage

### Kubernetes / Nodes

Node-level view:
- CPU, memory, disk, network utilization
- Pod capacity and count
- Node filesystem

### Kubernetes / API Server

API server health and performance:
- Request rate and latency
- etcd request duration
- Workqueue depth

### Node Exporter

Full OS metrics:
- CPU per core
- Memory breakdown
- Disk I/O
- Network interfaces

---

## Step 7: Useful PromQL Queries

Open the **Explore** tab in Grafana or use the Prometheus UI directly.

```
# CPU usage per pod (last 5 minutes)
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)

# Memory usage per pod
sum(container_memory_working_set_bytes{container!=""}) by (pod)

# Pod restarts
kube_pod_restarts_total

# Network receive bytes per pod
sum(rate(container_network_receive_bytes_total{container!=""}[5m])) by (pod)

# Available disk space per node
node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100

# Container filesystem usage per pod
sum(container_fs_usage_bytes{container!=""}) by (pod)
```

---

## Troubleshooting

### Pods stuck in ContainerCreating / ErrImagePull

In restricted network environments (e.g., China), container registries like `quay.io` and `docker.io` may be slow or unreachable. Pre-pull the required images:

```bash
# Find which images are needed
kubectl describe pod -n monitoring <pod-name> | grep -A5 "Failed"

# Pull them manually on each node
sudo crictl pull docker.io/grafana/grafana:11.5.2
sudo crictl pull quay.io/prometheus/prometheus:v3.2.1
sudo crictl pull quay.io/prometheus/node-exporter:v1.9.1
sudo crictl pull registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.15.0
```

Or configure a Docker registry mirror in `/etc/containers/registries.conf.d/`.

### Grafana shows no data

Check if Prometheus has the targets scraped:

```bash
# Port-forward to Prometheus UI
kubectl port-forward -n monitoring prometheus-kube-prometheus-kube-prome-prometheus-0 9090

# Visit http://localhost:9090/targets
```

Also verify kube-state-metrics is running — it's the source for pod/deployment/namespace object metrics.

### Can't access Grafana NodePort

If your firewall blocks the NodePort range (30000-32767), open it:

```bash
sudo firewall-cmd --permanent --add-port=30000-32767/tcp
sudo firewall-cmd --reload
```

Or use `kubectl port-forward` instead:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-grafana 3000:80
# Access at http://localhost:3000
```

---

## Integration With Other Services

### HAProxy

Add the HAProxy exporter to your Prometheus ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: haproxy
  namespace: monitoring
spec:
  endpoints:
  - port: stats
    interval: 30s
  selector:
    matchLabels:
      app: haproxy
```

### Ceph

Enable the Prometheus module on your Ceph cluster:

```bash
sudo ceph mgr module enable prometheus
```

Then add a ServiceMonitor or scrape config pointing to `http://<ceph-node>:9283/metrics`.

### Custom Applications

If your app exposes `/metrics`, just add these annotations to the pod template:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
```

Prometheus discovers it automatically.

---

## Uninstall

```bash
helm uninstall kube-prometheus -n monitoring
kubectl delete namespace monitoring
```

---

## Summary

| What | How it works |
|---|---|
| **Pod CPU/memory** | cAdvisor in kubelet, scraped automatically |
| **Pod state** | kube-state-metrics watches the API |
| **Node metrics** | node-exporter on every node |
| **Storage** | Prometheus TSDB on the Prometheus pod |
| **Visualization** | Grafana with pre-built dashboards |
| **Alerts** | Alertmanager handles routing |

The kube-prometheus-stack gives you production-grade monitoring with a single `helm install`. No manual scraping config, no dashboard building — it's ready out of the box.

---

*Installed via Helm on Kubernetes 1.31 with kube-prometheus-stack.*
