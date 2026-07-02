+++
date = '2026-07-02T00:36:00+08:00'
draft = false
title = '🖥️ CentOS 8 Server Initial Setup'
+++

## 📋 Overview

This post documents the initial configuration of a CentOS 8 VM (`ops-server`) — from default installation to a fully configured server.

## 📊 Baseline

| Item | Before | After |
|---|---|---|
| **Hostname** | `localhost.localdomain` | `ops-server.localdomain` |
| **IPv4** | DHCP (dynamic) | DHCP (with static attempt documented) |
| **IPv6** | Link-local only | `fd00::139/64` (static ULA) |
| **Sudo** | Password required | Passwordless for `jellyfish` |
| **Yum repos** | Dead mirrorlist | Aliyun vault mirror |
| **Shell** | Bash 4.x | Zsh 5.8 + Oh My Zsh |
| **Git** | Not installed | `git 2.27.0` |

---

## 1. 🔑 Passwordless Sudo

Edit sudoers to allow `jellyfish` to run commands without a password:

```bash
sudo visudo -f /etc/sudoers.d/jellyfish
```

Add this single line:

```
jellyfish ALL=(ALL) NOPASSWD: ALL
```

Always use `visudo` — it validates syntax before saving. A corrupted sudoers file can lock you out of the system. ⚠️

---

## 2. 🏷️ Hostname

```bash
sudo hostnamectl set-hostname ops-server.localdomain
```

Add to `/etc/hosts` for local resolution:

```bash
echo "127.0.0.1   ops-server.localdomain ops-server" | sudo tee -a /etc/hosts
```

---

## 3. 🌐 Network Configuration

### 3a. 🔍 Identify the Interface

```bash
nmcli con show
```

On this VM, the interface is `ens160`.

### 3b. 📡 Static IPv4 Attempt

The intended config was:

```bash
sudo nmcli con mod ens160 \
  ipv4.addresses 192.168.159.139/24 \
  ipv4.gateway 192.168.159.1 \
  ipv4.dns "192.168.159.2 8.8.8.8" \
  ipv4.method manual

sudo nmcli device reapply ens160
```

**❌ Problem:** The network gateway (192.168.159.1) blocked outbound traffic for statically-configured IPs — only DHCP-leased addresses could reach the internet. This is likely a firewall or gateway policy.

**✅ Fix:** Reverted to DHCP. For a "permanent static" effect, configure a **DHCP reservation** on the router — bind `192.168.159.139` to the VM's MAC address (`00:0c:29:ae:47:41`).

### 3c. 🌍 Static IPv6 (ULA)

Since no IPv6 router is available on this network, I configured a Unique Local Address (ULA):

```bash
sudo nmcli con mod ens160 \
  ipv6.addresses 'fd00::139/64' \
  ipv6.method manual

sudo nmcli device reapply ens160
```

This gives the server a reachable IPv6 address (`fd00::139`) for direct connections within the local network.

**Verify:**

```bash
ip -6 addr show ens160
# → inet6 fd00::139/64 scope global
```

---

## 4. 📦 Yum Repository Mirror

CentOS 8 reached EOL, so the default `mirrorlist.centos.org` is dead. 💀 I switched to the **Aliyun vault mirror** (reliable from China):

```bash
# Disable mirrorlist, enable Aliyun vault
for repo in /etc/yum.repos.d/CentOS-Linux-*.repo; do
  sudo sed -i 's/^mirrorlist/#mirrorlist/g' "$repo"
done

# Set baseurl for BaseOS and AppStream
sudo sed -i 's|^enabled=1|baseurl=https://mirrors.aliyun.com/centos-vault/8.5.2111/BaseOS/$basearch/os/\nenabled=1|' \
  /etc/yum.repos.d/CentOS-Linux-BaseOS.repo

sudo sed -i 's|^enabled=1|baseurl=https://mirrors.aliyun.com/centos-vault/8.5.2111/AppStream/$basearch/os/\nenabled=1|' \
  /etc/yum.repos.d/CentOS-Linux-AppStream.repo
```

Disable unused repos:

```bash
sudo sed -i 's/^enabled=1/enabled=0/' \
  /etc/yum.repos.d/CentOS-Linux-{Extras,FastTrack,Plus,Powertools}.repo 2>/dev/null
```

**Verify:**

```bash
sudo dnf clean all
sudo dnf makecache
```

**Other mirror options:**
- `vault.centos.org` — official archive
- `mirrors.aliyun.com/centos-vault/` — Alibaba Cloud (used here) 🇨🇳
- `mirrors.tuna.tsinghua.edu.cn/centos-vault/` — Tsinghua University
- `mirrors.ustc.edu.cn/centos-vault/` — USTC

---

## 5. 🐚 Zsh + Oh My Zsh

Zsh wasn't available in the CentOS 8 repos (all related packages were removed). Since sudo was configured, we could have used `dnf`, but instead I went with a **static build** for portability:

```bash
# Download static zsh binary (romkatv/zsh-bin)
curl -LO https://github.com/romkatv/zsh-bin/releases/download/v6.1.1/zsh-5.8-linux-x86_64.tar.gz
tar -xzf zsh-5.8-linux-x86_64.tar.gz
cp bin/zsh ~/.local/bin/
```

### Installing Oh My Zsh (without git)

Since git wasn't installed, I downloaded the archive directly:

```bash
curl -LO https://github.com/ohmyzsh/ohmyzsh/archive/master.tar.gz
tar -xzf master.tar.gz
cp -r ohmyzsh-master/* ~/.oh-my-zsh/
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```

### Fixing Zsh Functions (fpath)

The static zsh binary expects its function files at `/usr/local/share/zsh/5.8/functions/`, which doesn't exist. Extract them from the tarball and set a custom `FPATH`:

```bash
# Extract functions from zsh tarball
cp -r share/zsh/5.8/functions/* ~/.local/share/zsh/functions/

# Add to .zshrc (before Oh My Zsh loads)
export FPATH="$HOME/.local/share/zsh/functions:$FPATH"
```

### Plugins

```bash
# Install zsh-syntax-highlighting
git clone --depth 1 https://github.com/zsh-users/zsh-syntax-highlighting.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

# Install zsh-autosuggestions
git clone --depth 1 https://github.com/zsh-users/zsh-autosuggestions.git \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

Enable in `~/.zshrc`:

```bash
plugins=(
  git
  zsh-autosuggestions
  zsh-syntax-highlighting    # MUST be last
)
```

### Terminfo Fix 🛠️

If you see `can't find terminal definition for xterm-256color`:

```bash
sudo dnf install -y ncurses-term
```

---

## 6. 🔍 fzf — Fuzzy Finder

```bash
# Download binary
curl -LO https://github.com/junegunn/fzf/releases/download/v0.73.1/fzf-0.73.1-linux_amd64.tar.gz
tar -xzf fzf-0.73.1-linux_amd64.tar.gz
cp fzf ~/.local/bin/

# Install shell integration scripts
cp shell/completion.zsh ~/.fzf/shell/
cp shell/key-bindings.zsh ~/.fzf/shell/
```

Source in `~/.zshrc`:

```bash
[ -f ~/.fzf.zsh ] && source ~/.fzf.zsh
```

---

## 7. 📥 Install Git

With working yum repos, install git is trivial:

```bash
sudo dnf install -y git
```

---

## ✅ Final Verification

```bash
$ ssh jellyfish@ops-server
$ ~/.local/bin/zsh

# Oh My Zsh loaded
# Plugins active: git, syntax-highlighting, autosuggestions
# fzf bound to Ctrl+R / Ctrl+T / Alt+C
# Passwordless sudo: ✓
# Yum repos: Aliyun vault ✓
# IPv6: fd00::139 ✓
```

## 📊 Summary

| Component | Method | Status |
|---|---|---|
| Hostname | `hostnamectl` | ✅ |
| IPv4 | DHCP (reservation recommended) | ✅ |
| IPv6 | Static ULA `fd00::139/64` | ✅ |
| Sudo | `visudo` NOPASSWD | ✅ |
| Yum | Aliyun vault mirror | ✅ |
| Shell | Static Zsh 5.8 + Oh My Zsh | ✅ |
| Plugins | syntax-highlighting, autosuggestions | ✅ |
| fzf | v0.73.1 with key bindings | ✅ |
| Git | dnf install 2.27.0 | ✅ |
