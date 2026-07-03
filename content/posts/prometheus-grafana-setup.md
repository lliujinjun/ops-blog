+++
date = '2026-07-03T20:27:00+08:00'
draft = false
title = '📊 Prometheus & Node Exporter & Grafana Setup'
+++

## 📋 Overview

This post covers setting up the classic server monitoring stack: **Prometheus** for metrics storage, **Node Exporter** for collecting system metrics, and **Grafana** for dashboards.

The stack:

```
Node Exporter  ──►  Prometheus  ──►  Grafana
(metrics on       (collects &      (dashboards &
 each server)      stores data)     alerts)
```

---

## Step 1: 📡 Install Node Exporter

Node Exporter runs on **each server** you want to monitor. It exposes system metrics (CPU, memory, disk, network) on port `9100`.

```bash
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.8.2.linux-amd64.tar.gz
tar -xzf node_exporter-*.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/
```

Create a system user:

```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

Create a systemd service file at `/etc/systemd/system/node_exporter.service`:

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

Verify: `curl http://localhost:9100/metrics` — you should see a wall of metric data.

---

## Step 2: 📈 Install Prometheus

Prometheus runs on a **central server** and scrapes metrics from all Node Exporters.

```bash
wget https://github.com/prometheus/prometheus/releases/latest/download/prometheus-2.54.1.linux-amd64.tar.gz
tar -xzf prometheus-*.tar.gz
sudo mv prometheus-*/prometheus prometheus-*/promtool /usr/local/bin/
```

Create config at `/etc/prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets:
        - 'localhost:9100'            # this server
        - '192.168.8.128:9100'        # your first server
        # add more servers here
```

Create systemd service at `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

Verify: `curl http://localhost:9090` — Prometheus UI.

---

## Step 3: 📊 Install Grafana

Grafana provides the web dashboards. It connects to Prometheus as a data source.

```bash
# RHEL/CentOS/Fedora
sudo dnf install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-11.3.0-1.x86_64.rpm

# Ubuntu/Debian
sudo apt install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt update && sudo apt install -y grafana
```

Start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
```

Grafana UI at `http://<your-server-ip>:3000` — default login `admin` / `admin`.

### ⚙️ Configure Grafana

1. Log in — change the default password
2. Go to **Configuration → Data Sources → Add data source**
3. Select **Prometheus**
4. Set URL: `http://localhost:9090`
5. Click **Save & Test** — should show a green check

### 📋 Import a Dashboard

1. Go to **Dashboards → Import**
2. Enter dashboard ID: **1860** (Node Exporter Full)
3. Click **Load**
4. Select your Prometheus data source
5. Click **Import**

You should now see CPU, memory, disk, and network graphs for all your servers.

---

## 🔌 Port Reference

| Service | Port | Access |
|---------|------|--------|
| Node Exporter | 9100 | Internal (only Prometheus needs it) |
| Prometheus | 9090 | Admin UI (lock down or leave local) |
| Grafana | 3000 | Web dashboards |

To restrict Node Exporter access to Prometheus only:

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<prometheus-ip>" port protocol="tcp" port="9100" accept'
```

---

## 🧪 Testing

Once running, open Grafana → **Explore** → query `node_cpu_seconds_total` — you should see CPU metrics from your monitored servers.

---

## Summary

```
┌─────────────┐
│ Grafana     │  :3000  ← browse for charts
│ (dashboards)│
└──────┬──────┘
       │ queries PromQL
       ▼
┌─────────────┐
│ Prometheus  │  :9090  ← stores all metrics
│ (TSDB)      │
└──────┬──────┘
       │ scrapes /metrics every 15s
       ▼
┌─────────────┐   ┌─────────────┐
│ Node Exp.   │   │ Node Exp.   │
│ (server 1)  │   │ (server 2)  │
│ port 9100   │   │ port 9100   │
└─────────────┘   └─────────────┘
```

Three components. One unified dashboard. Ready for production monitoring. 🚀
