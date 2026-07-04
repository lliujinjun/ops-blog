+++
date = '2026-07-04T12:47:00+08:00'
draft = false
title = '🐍 Conda + Ansible: Install, Env, and Pack Export'
+++

## 📋 Overview

This post documents setting up Conda on a CentOS 8 server, creating a dedicated Ansible environment, and exporting it with `conda-pack` for redistribution.

Why not just `pip install ansible`? Conda environments are fully self-contained — they bundle Python itself, libffi, OpenSSL, and system-level dependencies. A packed conda env can be shipped to another machine (even offline) and activated instantly without any system package manager involvement.

---

## 0. Starting Point

A fresh CentOS 8 server (`192.168.8.128`) with Docker already installed. No Conda, no Python venv, no Ansible.

---

## 1. 🔽 Install Miniconda

```bash
$ cd /tmp
$ curl -sLO https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
$ sudo bash Miniconda3-latest-Linux-x86_64.sh -b -p /opt/conda
$ sudo chown -R jellyfish:jellyfish /opt/conda
```

The `-b` flag runs it silently (batch mode), `-p` sets the install path. I put it in `/opt/conda` so all users can access it.

### 🔥 TOS Surprise — Conda 26.x

First attempt to create an environment failed:

```
CondaToSNonInteractiveError: Terms of Service have not been accepted
for the following channels:
    - https://repo.anaconda.com/pkgs/main
    - https://repo.anaconda.com/pkgs/r
```

Starting with Conda 26.x, Anaconda now requires explicit TOS acceptance. Quick fix:

```bash
$ conda tos accept
  accepted Terms of Service for https://repo.anaconda.com/pkgs/main
  accepted Terms of Service for https://repo.anaconda.com/pkgs/r
```

One-time step. After that, environments create normally. ✅

---

## 2. 🌱 Create the Ansible Environment

```bash
$ conda create -y -n ansible python=3.12
$ conda install -y -n ansible -c conda-forge ansible
```

**Key details:**
- Python 3.12 from defaults channel
- Ansible 14.1.0 (core 2.21.1) from conda-forge
- Environment lives at `/opt/conda/envs/ansible`
- Totals ~75 MB of packages including OpenSSL, libgcc, PyYAML, Jinja2, cryptography

No `pip` involved — all packages come from conda channels with pre-compiled binaries. 🏋️

```bash
$ conda run -n ansible ansible --version
ansible [core 2.21.1]
  configured module search path = [...]
```

---

## 3. 📦 Pack the Environment with conda-pack

`conda-pack` creates a relocatable tarball of the entire conda environment. It can be extracted anywhere, no conda installation required on the target.

### Install conda-pack

```bash
$ conda install -y -n base -c conda-forge conda-pack
```

Note: install it **in the base environment**, not the target env. It needs to be available when packing, but isn't needed in the packed output.

### Pack it

```bash
$ conda pack -n ansible -o /tmp/ansible-env.tar.gz
Collecting packages...
Packing environment at '/opt/conda/envs/ansible' to '/tmp/ansible-env.tar.gz'
[########################################] | 100% Completed | 27.0s
```

**Result:** 151 MB tarball, 44,705 files, compressed in 27 seconds.

### What's inside?

```bash
$ tar tzf /tmp/ansible-env.tar.gz | head
bin/2to3
bin/2to3-3.12
bin/ansible
bin/ansible-community
bin/ansible-config
bin/ansible-connection
bin/ansible-console
bin/ansible-doc
bin/ansible-galaxy
bin/ansible-inventory
```

All Ansible entrypoints plus their transitive dependencies.

---

## 4. 🚚 Using the Packed Env Elsewhere

On a target machine (no conda needed):

```bash
$ mkdir -p ~/ansible-env
$ tar -xzf ansible-env.tar.gz -C ~/ansible-env
$ source ~/ansible-env/bin/activate
(ansible-env) $ ansible --version
```

Or use it directly:

```bash
$ ~/ansible-env/bin/ansible-playbook site.yml
```

This is especially useful for:
- 🖥️ **Air-gapped servers** — pack once, `scp` the tarball, extract and run
- 🐳 **Docker images** — layer the env into a slim image without conda
- ⚡ **CI/CD pipelines** — cache the tarball instead of re-installing every run

---

## 📊 Summary

| Step | Tool | Time | Result |
|---|---|---|---|
| Install Miniconda | `bash installer.sh` | ~30s | `/opt/conda` |
| Create ansible env | `conda create + install` | ~60s | Python 3.12, Ansible 14.1 |
| Pack environment | `conda pack` | 27s | 151 MB, 44,705 files |
| **Total** | | **~2 min** | **Portable Ansible env** 🎯 |

## 💡 Lessons Learned

1. **📜 Conda 26.x needs TOS acceptance** — `conda tos accept` before creating any env. This is new in recent versions.
2. **🏗️ conda-pack is not a conda subcommand** — Running `conda pack` fails because it doesn't register as a plugin. Use `/opt/conda/bin/conda-pack` directly or the `conda-pack` binary CLI.
3. **📦 Offline-ready** — The packed tarball is truly standalone. Binaries, Python, libc deps — everything is in there. No conda, no `pip install`, no internet needed on the target.
4. **🔐 Permission-aware** — Installing to `/opt/conda` requires `sudo`, but ownership should be changed back to a regular user for day-to-day operations.

---

*Next: time to write some Ansible playbooks. But that's a post for another day.* 📝
