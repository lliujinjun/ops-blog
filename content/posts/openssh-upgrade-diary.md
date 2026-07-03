+++
date = '2026-07-03T22:30:00+08:00'
draft = false
title = '🖧 OpenSSH 9.9 Upgrade Diary: RPM Building on CentOS 8'
+++

This is the story of upgrading OpenSSH from 8.0 to 9.9 on a CentOS 8 server at `192.168.8.128` — what broke, how it got fixed, and the 7 rounds it took.

---

## The Problem 🔍

A post-quantum warning every time I SSH'd into the server:

```
$ ssh 192.168.8.128
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded.
```

My WSL2 client runs OpenSSH 10.2 with post-quantum key exchange (NTRU Prime + X25519 hybrid), but the CentOS 8 server is stuck on OpenSSH 8.0 from 2019. The server doesn't understand the new algorithms, and the client warns about it.

The fix: upgrade the server to OpenSSH 9.9+.

---

## Round 1: `dnf upgrade` — Nothing to Do ❌

The simplest path:

```bash
$ sudo dnf upgrade openssh-server
Last metadata expiration check: 1:40:25 ago
Dependencies resolved.
Nothing to do.
```

CentOS 8 is EOL. No new packages coming. OpenSSH 8.0 is the final version in the official repos.

---

## Round 2: Build from Source — Too Easy?

```bash
$ wget https://github.com/openssh/openssh-portable/archive/refs/tags/V_9_9_P1.tar.gz
$ ./configure && make
```

Compiled without errors. But `make install` would scatter files outside the package manager's knowledge. For a cleaner approach: **build a proper RPM**.

---

## Round 3: Fix Repos + Install rpm-build 📦

CentOS 8 repos point to `mirror.centos.org` which no longer exists. First, switch to vault:

```bash
sudo sed -i 's/mirror.centos.org/vault.centos.org/g' /etc/yum.repos.d/CentOS-*
sudo sed -i 's/^#baseurl/baseurl/g' /etc/yum.repos.d/CentOS-*
sudo sed -i 's/^mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sudo dnf clean all
sudo dnf install -y rpm-build
```

Then download the CentOS SRPM for the spec file:

```bash
wget https://vault.centos.org/centos/8/BaseOS/Source/SPackages/openssh-8.0p1-10.el8.src.rpm
rpm -ivh openssh-8.0p1-10.el8.src.rpm
```

---

## Round 4: Spec File — Three Attempts 📜

The original spec has **65 patches** for OpenSSH 8.0 — none of them apply to 9.9. I wrote a minimal spec instead.

**Attempt 1:** `%files` section had incorrect paths. `%{_bindir}/sshd` doesn't exist — `sshd` is in `/usr/sbin/`, not `/usr/bin/`.

**Attempt 2:** Duplicate file listings and a missing `debugsourcefiles.list` caused RPM generation to fail.

**Attempt 3:** Added `%global debug_package %{nil}` to disable debuginfo, fixed all paths, listed `libexec` files explicitly. The spec finally built.

---

## Round 5: RPM Install — File Conflicts 💥

```bash
$ sudo rpm -Uvh openssh99-9.9-1.el8.x86_64.rpm
...
file /usr/sbin/sshd from install of openssh99-9.9-1.el8.x86_64
  conflicts with file from package openssh-server-8.0p1-10.el8.x86_64
file /usr/bin/ssh from install of openssh99-9.9-1.el8.x86_64
  conflicts with file from package openssh-clients-8.0p1-10.el8.x86_64
...
```

22 file conflicts. The old RPMs (`openssh`, `openssh-clients`, `openssh-server`) own the same paths. Can't remove them while connected via SSH. Used `--replacefiles` to override:

```bash
sudo rpm -Uvh --replacefiles ~/rpmbuild/RPMS/x86_64/openssh99-9.9-1.el8.x86_64.rpm
```

---

## Round 6: `sshd` Won't Start — Bad Config Options 🔥

```bash
$ sudo systemctl restart sshd
sshd[156004]: Bad configuration option: GSSAPIKexAlgorithms
```

OpenSSH 9.9 deprecated the `GSSAPIKexAlgorithms` option that was in the old config. Also removed all legacy GSSAPI options:

```bash
sudo sed -i '/GSSAPI/d' /etc/ssh/sshd_config
sudo sed -i '/GSSAPI/d' /etc/crypto-policies/back-ends/opensshserver.config
```

---

## Round 7: Host Key Permissions 🔑

After fixing GSSAPI, another issue:

```bash
$ sudo /usr/sbin/sshd -t
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0640 for '/etc/ssh/ssh_host_rsa_key' are too open.
```

The new `sshd` is stricter about permissions. Keys need to be `0600`:

```bash
sudo chmod 600 /etc/ssh/ssh_host_*_key
sudo systemctl restart sshd
```

**Active.** ✅

---

## The Result 🎯

```bash
$ ssh 192.168.8.128
# No PQ warning. Clean connection. With post-quantum protection.
```

| Round | Problem | Fix |
|-------|---------|-----|
| 1 | `dnf upgrade` has nothing to do | Can't — CentOS 8 EOL |
| 2 | Bare `make install` isn't clean | Build RPM instead |
| 3 | Repos are dead | Switch to vault.centos.org |
| 4 | Spec file broken 3 times | Write minimal spec, fix paths |
| 5 | File conflicts with old openssh RPMs | `--replacefiles` |
| 6 | GSSAPI options deprecated | `sed -i '/GSSAPI/d'` |
| 7 | Host key permissions too loose | `chmod 600` |

**7 rounds. One working SSH upgrade. Zero dropped connections.** 🖧
