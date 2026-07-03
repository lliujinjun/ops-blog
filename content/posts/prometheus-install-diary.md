+++
date = '2026-07-03T21:00:00+08:00'
draft = false
title = '📊 Prometheus Stack Install Diary: A Real Session on CentOS 8'
+++

This is a companion to the [Prometheus & Node Exporter & Grafana Setup]({{< relref "/posts/prometheus-grafana-setup" >}}) guide — what actually happened when I ran it on a CentOS 8 server at `192.168.8.128`.

The guide assumes **smooth sailing**. Real life? Not so much. 🌊

---

## 1. Node Exporter — Hit SELinux 💥

### Download — Smooth ✅

```bash
wget -q https://gh-proxy.com/https://github.com/prometheus/node_exporter/releases/download/v1.11.1/node_exporter-1.11.1.linux-amd64.tar.gz -O /tmp/node_exporter.tar.gz
tar -xzf /tmp/node_exporter.tar.gz -C /tmp
sudo mv /tmp/node_exporter-*/node_exporter /usr/local/bin/
```

No issues. Binary downloaded and moved into place.

### systemd Service — Crashed ❌

Created the service file, ran `sudo systemctl enable --now node_exporter`, and checked status:

```
● node_exporter.service - Node Exporter
   Loaded: loaded
   Active: failed (Result: exit-code)
  Process: 65317 ExecStart=/usr/local/bin/node_exporter (code=exited, status=203/EXEC)
```

Exit code **203/EXEC** — systemd can't execute the binary. But the file is right there:

```bash
$ file /usr/local/bin/node_exporter
ELF 64-bit LSB executable, x86-64, statically linked
```

The binary is fine. So what's wrong?

### Root Cause: SELinux 🔍

```bash
$ sealert -l <alert-id>
SELinux is preventing /usr/lib/systemd/systemd from execute access on the file node_exporter.
```

The binary was downloaded to `/tmp`, which has the SELinux label `user_tmp_t`. When moved to `/usr/local/bin/`, it **kept** the `user_tmp_t` label. SELinux sees it as a temp file being smuggled into a system location and blocks execution.

```
Target Context:  unconfined_u:object_r:user_tmp_t:s0  ← wrong label
Should be:       unconfined_u:object_r:bin_t:s0
```

### Fix: restorecon ✅

One command fixed it:

```bash
sudo restorecon -v /usr/local/bin/node_exporter
```

```
Relabeled /usr/local/bin/node_exporter from unconfined_u:object_r:user_tmp_t:s0 to unconfined_u:object_r:bin_t:s0
```

Restarted the service:

```bash
sudo systemctl restart node_exporter
sudo systemctl is-active node_exporter
# → active ✅
```

curl localhost:9100/metrics confirmed metrics flowing. Took about 10 minutes to debug. Note to future self: **always `restorecon` after moving binaries from `/tmp`**. 🛠️

---

## 2. Prometheus — Just Worked ✅

### Download

```bash
wget -q https://github.com/prometheus/prometheus/releases/download/v3.13.0/prometheus-3.13.0.linux-amd64.tar.gz -O /tmp/prometheus.tar.gz
```

100 MB download, took about 30 seconds.

### Extract & Install

```bash
tar -xzf /tmp/prometheus.tar.gz -C /tmp
sudo mv /tmp/prometheus-*/prometheus /tmp/prometheus-*/promtool /usr/local/bin/
```

No surprises. Created the config and systemd service:

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets:
        - localhost:9100
```

Enabled and started:

```bash
sudo systemctl enable --now prometheus
```

Checked:

```bash
$ curl http://localhost:9090
# → Prometheus UI reachable ✅
```

Zero errors. No SELinux issues. Prometheus 3.13.0 just ran out of the box. 🤷

---

## 3. Grafana — Just Worked ✅

```bash
sudo dnf install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-11.3.0-1.x86_64.rpm
sudo systemctl enable --now grafana-server
```

Installed, started, login page at port 3000. `admin`/`admin`, change password, add Prometheus data source, import dashboard ID **1860**. Done.

---

## 📊 Summary

| Component | Smooth | Drama | Fix |
|-----------|--------|-------|-----|
| Node Exporter | ❌ | SELinux blocked exec on `user_tmp_t` | `restorecon` |
| Prometheus | ✅ | Nothing | — |
| Grafana | ✅ | Nothing | — |

**Lesson learned:** Always run `restorecon -v /usr/local/bin/*` after manually extracting binaries from `/tmp`. SELinux doesn't care that you moved the file — it cares about the **original** security context. 🧠
