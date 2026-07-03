+++
date = '2026-07-03T23:30:00+08:00'
draft = false
title = '🖧 Upgrading OpenSSH on CentOS 8 via RPM Build'
+++

## 📋 Overview

This guide covers upgrading OpenSSH on CentOS 8 from the stock 8.0p1 to the latest 9.9p1 by building a custom RPM. This is useful when:

- ⚠️ Your SSH client warns about missing post-quantum key exchange algorithms
- 🔒 You need modern SSH features on an EOL CentOS 8 system
- 📦 You want to maintain package manager tracking (vs. bare `make install`)

✅ Tested on CentOS 8.5 at `192.168.8.128`.

---

## Step 1: 🛠️ Install Build Tools

```bash
sudo dnf install -y rpm-build gcc make
sudo dnf install -y openssl-devel pam-devel zlib-devel
```

---

## Step 2: 📦 Set Up RPM Build Tree & Download Sources

```bash
mkdir -p ~/rpmbuild/{SPECS,SOURCES}

# 📥 Download the CentOS 8 SRPM (for reference, not used directly)
wget -P /tmp https://vault.centos.org/centos/8/BaseOS/Source/SPackages/openssh-8.0p1-10.el8.src.rpm
rpm -ivh /tmp/openssh-8.0p1-10.el8.src.rpm

# 🌐 Download OpenSSH 9.9 portable source
wget -O ~/rpmbuild/SOURCES/openssh-9.9.tar.gz \
  https://github.com/openssh/openssh-portable/archive/refs/tags/V_9_9_P1.tar.gz
```

> 💡 The SRPM provides the spec file structure. But it has **65 patches** for 8.0 that won't apply to 9.9 — we write our own spec.

---

## Step 3: 📄 Write a Minimal Spec File

```bash
cat > ~/rpmbuild/SPECS/openssh99.spec << 'ENDSPEC'
Summary: OpenSSH 9.9p1
Name: openssh99
Version: 9.9
Release: 1%{?dist}
License: BSD
Source0: openssh-9.9.tar.gz
URL: https://www.openssh.com/
BuildRequires: openssl-devel, zlib-devel, pam-devel
BuildRequires: gcc, make
%global debug_package %{nil}

%description
OpenSSH 9.9p1 portable release.

%prep
%setup -q -n openssh-portable-V_9_9_P1

%build
%configure --prefix=/usr --sysconfdir=/etc/ssh --with-pam
make %{?_smp_mflags}

%install
%make_install

%files
/usr/bin/ssh
/usr/bin/scp
/usr/bin/ssh-add
/usr/bin/ssh-agent
/usr/bin/ssh-keygen
/usr/bin/ssh-keyscan
/usr/bin/sftp
/usr/sbin/sshd
/etc/ssh/moduli
/etc/ssh/ssh_config
/etc/ssh/sshd_config
/usr/libexec/sshd-session
/usr/libexec/ssh-keysign
/usr/libexec/ssh-pkcs11-helper
/usr/libexec/ssh-sk-helper
/usr/libexec/sftp-server
/usr/share/man/man1/*.gz
/usr/share/man/man5/*.gz
/usr/share/man/man8/*.gz

%post
systemctl daemon-reload || true

%preun
if [ $1 -eq 0 ]; then systemctl stop sshd || true; fi

%postun
if [ $1 -ge 1 ]; then systemctl try-restart sshd || true; fi
ENDSPEC
```

> 💡 `%global debug_package %{nil}` disables debuginfo subpackages to avoid empty `debugsourcefiles.list` errors.

Key points:
- 🎯 Only requires `openssl-devel`, `zlib-devel`, `pam-devel` — no need for dozens of optional deps
- 🗺️ `%files` lists exact paths — avoids accidental omissions
- 🧹 No patches from the original spec — clean build from upstream source

---

## Step 4: 🔨 Build the RPM

```bash
cd ~/rpmbuild
rpmbuild -ba SPECS/openssh99.spec
```

⏱️ Takes about 1-2 minutes on a 2-core server.

✅ If successful, the RPM will be at `~/rpmbuild/RPMS/x86_64/openssh99-9.9-1.el8.x86_64.rpm`.

---

## Step 5: 📥 Install the New RPM

The new RPM conflicts with existing `openssh`, `openssh-clients`, and `openssh-server` packages. Use `--replacefiles`:

```bash
sudo rpm -Uvh --replacefiles ~/rpmbuild/RPMS/x86_64/openssh99-9.9-1.el8.x86_64.rpm
```

⚠️ This replaces files owned by the old RPMs without removing them. The old packages stay installed but their files are overwritten by 9.9's versions.

---

## Step 6: 🔧 Fix Deprecated Crypto Policy

OpenSSH 9.9 removed the `GSSAPIKexAlgorithms` option that CentOS 8's system crypto policy includes. Move the generated policy file aside:

```bash
sudo mv /etc/crypto-policies/back-ends/opensshserver.config \
       /etc/crypto-policies/back-ends/opensshserver.config.bak
```

> 💡 This removes the `CRYPTO_POLICY` override and deprecated GSSAPI options together. Safe because OpenSSH 9.9 has modern defaults built-in. The system `CRYPTO_POLICY` line is no longer needed.

---

## Step 7: 🔑 Fix Host Key Permissions

The new sshd is stricter about key file permissions:

```bash
sudo chmod 600 /etc/ssh/ssh_host_*_key
```

🔍 Why: The old sshd accepted `0640` perms. The new one requires `0600` — private keys must not be readable by group or others.

---

## Step 8: 🔄 Regenerate Host Keys (If Missing)

If `sshd -t` reports "no hostkeys available":

```bash
sudo ssh-keygen -A
```

This creates all missing host key types (`rsa`, `ecdsa`, `ed25519`).

💡 Run this if the upgrade is on a fresh system or if keys were lost during migration.

---

## Step 9: ✅ Restart and Verify

```bash
sudo systemctl restart sshd
ssh -V
```

Expected output:

```
OpenSSH_9.9p1, OpenSSL 1.1.1k  FIPS 25 Mar 2021
```

From your client machine, connect without the post-quantum warning:

```bash
ssh user@centos8-server
# ✅ No PQ warning. Clean connection.
```

---

## 📝 Quick Reference

```bash
# 1️⃣ Install build tools
sudo dnf install -y rpm-build gcc make openssl-devel pam-devel zlib-devel

# 2️⃣ Download sources
mkdir -p ~/rpmbuild/{SPECS,SOURCES}
wget -P /tmp https://vault.centos.org/centos/8/BaseOS/Source/SPackages/openssh-8.0p1-10.el8.src.rpm
rpm -ivh /tmp/openssh-8.0p1-10.el8.src.rpm
wget -O ~/rpmbuild/SOURCES/openssh-9.9.tar.gz \
  https://github.com/openssh/openssh-portable/archive/refs/tags/V_9_9_P1.tar.gz

# 3️⃣ Write spec (paste from Step 3 above)

# 4️⃣ Build
cd ~/rpmbuild && rpmbuild -ba SPECS/openssh99.spec

# 5️⃣ Install
sudo rpm -Uvh --replacefiles ~/rpmbuild/RPMS/x86_64/openssh99-9.9-1.el8.x86_64.rpm

# 6️⃣ Fix crypto policy
sudo mv /etc/crypto-policies/back-ends/opensshserver.config \
       /etc/crypto-policies/back-ends/opensshserver.config.bak

# 7️⃣ Fix key perms
sudo chmod 600 /etc/ssh/ssh_host_*_key

# 8️⃣ Regenerate keys (if needed)
sudo ssh-keygen -A

# 9️⃣ Restart
sudo systemctl restart sshd
ssh -V
```

---

## 🧪 Validation

```
Before:                          After:
ssh user@centos8-server          ssh user@centos8-server
⚠️ WARNING: PQ key exchange      ✅ Clean connection
  "store now, decrypt later"        OpenSSH 9.9 on server
```
