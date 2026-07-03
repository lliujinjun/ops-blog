+++
date = '2026-07-04T00:25:00+08:00'
draft = false
title = '⚙️ Enabling cgroup v2 on CentOS 8 with Modern Kernel'
+++

## 📋 Overview

This guide covers enabling **cgroup v2** (unified control group hierarchy) on CentOS 8 after upgrading to a modern kernel via ELRepo.

cgroup v2 is the current standard for Linux resource management and is required for modern container runtimes like rootless Docker and Podman 4+.

---

## Check Current cgroup Version

```bash
stat -fc %T /sys/fs/cgroup/
```

Possible outputs:

| Output | Version | Meaning |
|--------|---------|---------|
| `tmpfs` | **cgroup v1** | Default on CentOS 8 stock kernel |
| `cgroup2fs` | **cgroup v2** | Unified hierarchy |

On a stock CentOS 8.5 with kernel `4.18.0-348.el8`, you'll get `tmpfs` — you're on cgroup v1.

---

## Why Upgrade to cgroup v2?

- 🐳 **Rootless Docker** — requires cgroup v2 (or special workarounds)
- 📦 **Podman 4+** — uses cgroup v2 by default for better isolation
- 📊 **Pressure Stall Information (PSI)** — better memory/IO pressure monitoring
- 🔧 **Unified hierarchy** — single tree for CPU, memory, IO instead of multiple controllers
- 🚀 **Performance** — lighter weight than v1's multiple hierarchies

⚠️ **Caveat:** Some legacy monitoring tools (older versions of `systemd-cgtop`, `docker stats`) may not display full data under cgroup v2. Check your tooling first.

---

## Prerequisites

You need a kernel that supports cgroup v2. Any kernel 4.15+ supports it. If you're still on the stock CentOS 8 kernel, upgrade first:

> See the [Kernel Upgrade via ELRepo]({{< relref "/posts/kernel-upgrade-elrepo" >}}) guide.

---

## Step 1: Enable cgroup v2 via Kernel Parameter

Add `systemd.unified_cgroup_hierarchy=1` to the kernel boot command line:

```bash
sudo grubby --args="systemd.unified_cgroup_hierarchy=1" --update-kernel DEFAULT
```

Verify it was added:

```bash
sudo grubby --info DEFAULT | grep args
```

Expected output:

```
args="ro crashkernel=auto systemd.unified_cgroup_hierarchy=1 ..."
```

---

## Step 2: Reboot

```bash
sudo reboot
```

---

## Step 3: Verify

After reboot:

```bash
stat -fc %T /sys/fs/cgroup/
```

Expected:

```
cgroup2fs
```

Also check:

```bash
ls -la /sys/fs/cgroup/ | head -10
```

Under v2, you'll see a flat structure with files like `cgroup.controllers`, `cgroup.procs`, `cpu.max`, `memory.max` — no separate subdirectories for each controller.

---

## Step 4: Update Container Runtime (If Applicable)

### Docker

```bash
sudo systemctl edit docker
```

Add:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --exec-opt native.cgroupdriver=systemd
```

Reload:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
docker info | grep cgroup
```

Expected:

```
Cgroup Driver: systemd
Cgroup Version: 2
```

### Podman

Podman 4+ auto-detects cgroup v2. No configuration needed.

---

## Rolling Back

If you encounter issues:

1. Remove the parameter:
   ```bash
   sudo grubby --remove-args="systemd.unified_cgroup_hierarchy=1" --update-kernel DEFAULT
   ```
2. Reboot back into cgroup v1.

---

## 📝 Quick Reference

```bash
# 1️⃣ Check current version
stat -fc %T /sys/fs/cgroup/

# 2️⃣ Enable cgroup v2
sudo grubby --args="systemd.unified_cgroup_hierarchy=1" --update-kernel DEFAULT
sudo reboot

# 3️⃣ Verify
stat -fc %T /sys/fs/cgroup/

# 4️⃣ Rollback
sudo grubby --remove-args="systemd.unified_cgroup_hierarchy=1" --update-kernel DEFAULT
sudo reboot
```

---

## 🧪 Validation

```
Before:                          After:
stat -fc %T /sys/fs/cgroup/      stat -fc %T /sys/fs/cgroup/
tmpfs                             cgroup2fs
```
