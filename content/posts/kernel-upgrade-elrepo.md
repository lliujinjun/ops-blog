+++
date = '2026-07-04T00:15:00+08:00'
draft = false
title = '🐧 Upgrading the Kernel on CentOS 8 via ELRepo'
+++

## 📋 Overview

This guide covers upgrading the kernel on CentOS 8 from the stock `4.18.0-348.el8` to a newer version via [ELRepo](http://elrepo.org/). This is useful when:

- 🚀 You need modern kernel features (BBR congestion control, io_uring, better hardware support)
- 🔒 Newer kernels have better security hardening
- 🖥️ You're hitting driver or filesystem limits on the stock kernel

---

## Which Kernel to Pick?

ELRepo offers two options:

| Option | Version at time of writing | Best For |
|--------|---------------------------|----------|
| **kernel-ml** (Mainline) | `7.1.2-1.el8.elrepo` | Latest features, newer hardware |
| **kernel-lt** (LTS) | `5.15.210-1.el8.elrepo` | Stability, production servers |

💡 **Recommendation:** For a production server, start with **kernel-lt**. It's stable, well-tested, and receives long-term security backports. Use **kernel-ml** if you need a specific newer feature.

---

## Step 1: Install ELRepo

```bash
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo dnf install -y https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm
```

Verify the repo is enabled:

```bash
sudo dnf repolist | grep elrepo
```

---

## Step 2: Install the New Kernel

### Option A: Mainline Kernel (kernel-ml)

```bash
sudo dnf --enablerepo=elrepo-kernel install -y kernel-ml
```

### Option B: LTS Kernel (kernel-lt)

```bash
sudo dnf --enablerepo=elrepo-kernel install -y kernel-lt
```

Both install the kernel alongside your existing one — the old kernel is preserved as a fallback.

---

## Step 3: Set the New Kernel as Default

Check what was installed:

```bash
rpm -qa kernel-ml kernel-lt
```

Find the installed kernel:

```bash
ls /boot/vmlinuz-*
```

Set it as the default boot entry:

```bash
sudo grubby --set-default /boot/vmlinuz-7.1.2-1.el8.elrepo.x86_64
```

> Replace `7.1.2` with the actual kernel version installed.

Verify:

```bash
sudo grubby --default-kernel
```

---

## Step 4: Reboot

```bash
sudo reboot
```

After the server comes back, verify:

```bash
uname -r
```

Expected output (kernel-ml):

```
7.1.2-1.el8.elrepo.x86_64
```

Or (kernel-lt):

```
5.15.210-1.el8.elrepo.x86_64
```

---

## Step 5: Clean Up Old Kernels (Optional)

After confirming the new kernel works, remove old kernels:

```bash
sudo dnf remove -y $(dnf repoquery --installonly --latest-limit=-1 -q)
```

Or manually:

```bash
sudo dnf remove -y kernel-core-4.18.0-348.el8
```

> ⚠️ Keep at least one backup kernel in case of boot issues.

---

## Troubleshooting

### Booting into the old kernel after reboot

Check the grub default:

```bash
sudo grubby --default-kernel
```

If it's still the old one, re-run Step 3 and reboot again.

### Server won't boot with the new kernel

At the GRUB menu during boot, select the old kernel. Once back in, reinstall:

```bash
sudo dnf reinstall -y kernel-ml
sudo grubby --set-default /boot/vmlinuz-7.1.2-1.el8.elrepo.x86_64
```

### Missing modules after upgrade

Kernel modules from the stock kernel aren't compatible with the new one. Rebuild any third-party modules (e.g., DKMS drivers):

```bash
sudo dkms autoinstall
```

---

## 📝 Quick Reference

```bash
# 1️⃣ Install ELRepo
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo dnf install -y https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm

# 2️⃣ Install new kernel (pick one)
sudo dnf --enablerepo=elrepo-kernel install -y kernel-ml    # mainline
sudo dnf --enablerepo=elrepo-kernel install -y kernel-lt    # LTS

# 3️⃣ Set default and reboot
sudo grubby --set-default /boot/vmlinuz-7.1.2-1.el8.elrepo.x86_64
sudo reboot

# 4️⃣ Verify
uname -r
```

---

## 🧪 Validation

```
Before:                          After:
uname -r                         uname -r
4.18.0-348.el8.x86_64           7.1.2-1.el8.elrepo.x86_64
```
