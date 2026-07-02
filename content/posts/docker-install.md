+++
date = '2026-07-02T14:59:00+08:00'
draft = false
title = 'Docker Engine Installation on RHEL/CentOS/Fedora'
+++

## Overview

This guide covers installing Docker Engine on RHEL-family Linux servers from the official Docker repository.

**What you'll get:**

| Component | Purpose |
|---|---|
| `docker-ce` | The Docker daemon (dockerd) |
| `docker-ce-cli` | The `docker` CLI command |
| `containerd.io` | Container runtime layer |
| `docker-buildx-plugin` | Multi-architecture image builds |
| `docker-compose-plugin` | Docker Compose v2 (`docker compose`) |

## Prerequisites

- A Linux server running RHEL 9+, CentOS Stream, or Fedora 38+
- Sudo or root access
- Internet connectivity to reach `download.docker.com`

---

## Step 1: Remove Old Versions

Older Docker packages (from EPEL, CentOS Extras, or manual installs) can conflict with the official release. Remove them cleanly:

```bash
sudo dnf remove -y docker docker-client docker-client-latest \
  docker-common docker-latest docker-latest-logrotate \
  docker-logrotate docker-engine podman runc
```

Don't worry if some return `not installed` — that's expected.

---

## Step 2: Add the Docker Repository

Install the `dnf-plugins-core` package (provides `config-manager`), then add the official Docker CE repo:

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

This writes the repo file to `/etc/yum.repos.d/docker-ce.repo`.

> **For users in China / slow networks:** Replace the repo URL with a mirror:
> ```bash
> sudo dnf config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/rhel/docker-ce.repo
> ```

---

## Step 3: Install Docker Engine

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

If prompted to accept the GPG key, verify the fingerprint matches Docker's official key (`9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`) and type **y**.

---

## Step 4: Start and Enable Docker

```bash
sudo systemctl enable --now docker
```

Check the status:

```bash
sudo systemctl status docker
```

Expected output includes `active (running)` and no obvious errors.

---

## Step 5: Run Docker as a Non-Root User

Running `docker` with `sudo` every time gets old. Add your user to the `docker` group:

```bash
sudo usermod -aG docker $USER
```

**Important:** Log out and back in (or run `newgrp docker` in the current shell) for the group change to take effect.

---

## Step 6: Verification

Run the Hello World container:

```bash
docker run hello-world
```

You should see:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

The first run pulls the `hello-world` image from Docker Hub — expect a few seconds.

---

## Step 7: Configure a Registry Mirror (Optional)

If Docker Hub pulls are slow, set up a registry mirror in `/etc/docker/daemon.json`:

```json
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
```

Then restart Docker:

```bash
sudo systemctl restart docker
```

Common mirrors:

| Mirror | URL |
|---|---|
| USTC (China) | `https://docker.mirrors.ustc.edu.cn` |
| Aliyun (China) | `https://<your-id>.mirror.aliyuncs.com` (requires Aliyun account) |
| Docker Hub (default) | `https://registry-1.docker.io` |

---

## Troubleshooting

| Issue | Fix |
|---|---|
| `Cannot connect to the Docker daemon` | Is `dockerd` running? `sudo systemctl start docker` |
| `permission denied` trying to connect | You forgot to log out/in after `usermod -aG docker` |
| GPG key import fails | Try `sudo dnf install -y dnf-plugins-core && sudo dnf config-manager --save --setopt=docker-ce-stable.gpgcheck=0` |

---

## Summary

```bash
# Clean install, three commands:
sudo dnf remove -y docker* podman runc           # clean slate
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Start it:
sudo systemctl enable --now docker

# Add your user:
sudo usermod -aG docker $USER   # then log out & back in

# Test it:
docker run hello-world
```
