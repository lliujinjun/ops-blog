+++
date = '2026-07-03T21:00:00+08:00'
draft = false
title = '🖧 Upgrading OpenSSH on CentOS 8 via RPM Build'
+++

## 📋 Overview

This guide covers upgrading OpenSSH on CentOS 8 from the stock 8.0p1 to the latest 9.9p1 by building a custom RPM. This is useful when:

- Your SSH client warns about missing post-quantum key exchange algorithms
- You need modern SSH features on an EOL CentOS 8 system
- You want to maintain package manager tracking (vs. `make install`)

Tested on CentOS 8.5 at `192.168.8.128`.

---

## Step 1: Fix the Repos

CentOS 8 is EOL — the default mirror URLs no longer resolve. Switch to `vault.centos.org`:

```bash
sudo sed -i 's/mirror.centos.org/vault.centos.org/g' /etc/yum.repos.d/CentOS-*
sudo sed -i 's/^#baseurl/baseurl/g' /etc/yum.repos.d/CentOS-*
sudo sed -i 's/^mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sudo dnf clean all
```

---

## Step 2: Install Build Tools

```bash
sudo dnf install -y rpm-build gcc make
sudo dnf install -y openssl-devel pam-devel zlib-devel
```

---

## Step 3: Set Up RPM Build Tree & Download Sources

```bash
mkdir -p ~/rpmbuild/{SPECS,SOURCES}

# Download the CentOS 8 SRPM (for reference, not used directly)
wget -P /tmp https://vault.centos.org/centos/8/BaseOS/Source/SPackages/openssh-8.0p1-10.el8.src.rpm
rpm -ivh /tmp/openssh-8.0p1-10.el8.src.rpm

# Download OpenSSH 9.9 portable source
wget -O ~/rpmbuild/SOURCES/openssh-9.9.tar.gz \
  https://github.com/openssh/openssh-portable/archive/refs/tags/V_9_9_P1.tar.gz
```

---

## Step 4: Write a Minimal Spec File

The original CentOS spec has 65 patches that only apply to 8.0. Instead, write a clean spec:

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

> `%global debug_package %{nil}` disables debuginfo subpackage generation to avoid empty file list errors.

---

## Step 5: Build the RPM

```bash
cd ~/rpmbuild
rpmbuild -ba SPECS/openssh99.spec
```

If successful, the RPM will be at `~/rpmbuild/RPMS/x86_64/openssh99-9.9-1.el8.x86_64.rpm`.

---

## Step 6: Install the New RPM

The new RPM conflicts with existing `openssh`, `openssh-clients`, and `openssh-server` packages. Use `--replacefiles`:

```bash
sudo rpm -Uvh --replacefiles ~/rpmbuild/RPMS/x86_64/openssh99-9.9-1.el8.x86_64.rpm
```

---

## Step 7: Fix Deprecated Config Options

OpenSSH 9.9 has removed legacy GSSAPI options. Clean up both the main config and the system crypto policy:

```bash
sudo sed -i '/GSSAPI/d' /etc/ssh/sshd_config
sudo sed -i '/GSSAPI/d' /etc/crypto-policies/back-ends/opensshserver.config
```

---

## Step 8: Regenerate Host Keys (If Missing)

If `sshd -t` reports "no hostkeys available", regenerate them:

```bash
sudo ssh-keygen -A
```

This creates all missing host key types (`rsa`, `ecdsa`, `ed25519`).

---

## Step 9: Restart and Verify

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
# No PQ warning. Clean connection. ✅
```

---

## Summary

```bash
# Quick reference — all steps in order:
sudo sed -i 's/mirror.centos.org/vault.centos.org/g' /etc/yum.repos.d/CentOS-*
sudo dnf install -y rpm-build gcc make openssl-devel pam-devel zlib-devel
mkdir -p ~/rpmbuild/{SPECS,SOURCES}
wget -P /tmp https://vault.centos.org/centos/8/BaseOS/Source/SPackages/openssh-8.0p1-10.el8.src.rpm
rpm -ivh /tmp/openssh-8.0p1-10.el8.src.rpm
wget -O ~/rpmbuild/SOURCES/openssh-9.9.tar.gz https://github.com/openssh/openssh-portable/archive/refs/tags/V_9_9_P1.tar.gz
# (write the spec file from Step 4)
cd ~/rpmbuild && rpmbuild -ba SPECS/openssh99.spec
sudo rpm -Uvh --replacefiles ~/rpmbuild/RPMS/x86_64/openssh99-9.9-1.el8.x86_64.rpm
sudo sed -i '/GSSAPI/d' /etc/ssh/sshd_config
sudo sed -i '/GSSAPI/d' /etc/crypto-policies/back-ends/opensshserver.config
sudo ssh-keygen -A
sudo systemctl restart sshd
ssh -V
```
