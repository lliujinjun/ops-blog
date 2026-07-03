+++
date = '2026-07-04T01:35:00+08:00'
draft = false
title = '📦 Harbor Installation Guide: Self-Hosted Container Registry'
+++

## 📋 Overview

Harbor is an open-source container image registry with built-in security scanning, replication, and role-based access control. Think Docker Hub but self-hosted on your own infrastructure.

This guide covers installing Harbor on a Linux server using the offline installer.

---

## Architecture

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Client   │   │  Harbor  │   │  Docker  │   │  Redis   │
│ (docker  │──►│  Portal  │──►│ Registry │──►│  (cache) │
│  push)   │   │ (UI/API) │   │ (images) │   └──────────┘
└──────────┘   └────┬─────┘   └────┬─────┘
                    │              │
                    ▼              ▼
             ┌──────────┐   ┌──────────┐
             │  DB      │   │  Job     │
             │(Postgres)│   │ Service  │
             └──────────┘   └──────────┘
```

---

## Prerequisites

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Disk | 40 GB | 100 GB+ (for images) |
| Docker | 20.10+ | Latest |
| Docker Compose | 1.29+ | v2 |
| OS | CentOS 8 / Ubuntu 22.04 | Any modern Linux |

Also needed:
- 🐳 Docker Engine installed and running
- 🔐 A domain name or public IP (for HTTPS)
- 📜 SSL certificate (Let's Encrypt or self-signed)

---

## Step 1: Install Docker & Docker Compose

If Docker isn't installed yet:

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker
```

---

## Step 2: Download Harbor

```bash
cd /opt
sudo wget https://github.com/goharbor/harbor/releases/latest/download/harbor-offline-installer-v2.12.0.tgz
sudo tar -xzf harbor-offline-installer-*.tgz
cd harbor
```

> 💡 The offline installer includes all Docker images needed — no internet access required during installation.

---

## Step 3: Configure Harbor

```bash
sudo cp harbor.yml.tmpl harbor.yml
sudo vi harbor.yml
```

Key settings to change:

```yaml
hostname: registry.example.com
http:
  port: 80
https:
  port: 443
  certificate: /data/cert/server.crt
  private_key: /data/cert/server.key
harbor_admin_password: ChangeMe123!
database:
  password: dbpass123
```

> ⚠️ For testing without a real domain, set `hostname` to the server's IP address and comment out the HTTPS section.

---

## Step 4: Run the Installer

```bash
sudo ./install.sh
```

The installer will load Docker images, start all containers via Docker Compose, and verify each service is healthy.

Expected output:

```
[✔] preparing network ... done
[✔] starting Harbor ... done
--- Harbor has been installed ---
```

---

## Step 5: Access the Web UI

```
https://registry.example.com
```

- **Username:** `admin`
- **Password:** (set in `harbor.yml`)

First thing: **change the admin password** — Admin → Settings.

---

## Step 6: Push Your First Image

```bash
docker login registry.example.com
docker tag nginx:latest registry.example.com/library/nginx:latest
docker push registry.example.com/library/nginx:latest
```

For HTTP-only setups:

```json
# /etc/docker/daemon.json
{ "insecure-registries": ["registry.example.com"] }
```

```bash
sudo systemctl restart docker
```

---

## Step 7: Enable HTTPS (Let's Encrypt)

```bash
sudo dnf install -y certbot
sudo certbot certonly --standalone -d registry.example.com
sudo mkdir -p /data/cert
sudo cp /etc/letsencrypt/live/registry.example.com/fullchain.pem /data/cert/server.crt
sudo cp /etc/letsencrypt/live/registry.example.com/privkey.pem /data/cert/server.key
sudo chmod 600 /data/cert/server.key
sudo docker-compose down && sudo ./install.sh
```

---

## 📝 Quick Reference

```bash
# Download & install
cd /opt && sudo wget https://github.com/goharbor/harbor/releases/latest/download/harbor-offline-installer-v2.12.0.tgz
sudo tar -xzf harbor-offline-installer-*.tgz && cd harbor
sudo cp harbor.yml.tmpl harbor.yml && sudo vi harbor.yml
sudo ./install.sh

# Test
docker login registry.example.com
docker tag nginx registry.example.com/library/nginx
docker push registry.example.com/library/nginx
```

---

## 🧪 Validation

```bash
curl -k https://registry.example.com/api/v2.0/ping
# → {"ping":"pong"}
```
