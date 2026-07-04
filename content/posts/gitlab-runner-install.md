+++
date = '2026-07-04T15:43:00+08:00'
draft = false
title = '🏃 GitLab Runner Installation on CentOS Stream 9'
+++

## 📋 Overview

GitLab Runner is the worker process that executes CI/CD jobs. This post covers installing it alongside GitLab on the same CentOS Stream 9 server — plus the same Tsinghua mirror detour for a smooth download.

---

## Step 1: Add the Repository

The official runner repo URL is for `el/9/x86_64`, but on Tsinghua mirror the path structure is different:

```bash
# Official (slow from China):
baseurl=https://packages.gitlab.com/runner/gitlab-runner/el/9/x86_64

# Tsinghua mirror (fast):
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-runner/yum/el9-x86_64/
```

The Tsinghua path uses `el9-x86_64` as a single directory name, not `el/9/x86_64`.

```bash
sudo tee /etc/yum.repos.d/gitlab-runner.repo > /dev/null <<"EOF"
[gitlab-runner]
name=gitlab-runner
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-runner/yum/el9-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
EOF
```

---

## Step 2: Install the Runner

```bash
sudo dnf install -y gitlab-runner
```

This downloads two packages:

| Package | Size | Description |
|---|---|---|
| `gitlab-runner-19.1.1-1.x86_64.rpm` | 29 MB | The runner binary |
| `gitlab-runner-helper-images-19.1.1-1.noarch.rpm` | 515 MB | Pre-built helper Docker images |

Total: **544 MB** — downloaded in about a minute from Tsinghua.

### 🔥 Potential Issues

If `dnf install` gets killed mid-way (SSH timeout), the RPMs may be downloaded but not installed. Check:

```bash
$ which gitlab-runner
/usr/bin/gitlab-runner

$ gitlab-runner --version
Version:      19.1.1
Git revision: 24b9b726
GO version:   go1.26.3
Built:        2026-06-25T11:31:53Z
OS/Arch:      linux/amd64
```

If the binary is missing, install from the cached RPM:

```bash
sudo rpm -ivh /var/cache/dnf/gitlab-runner-*/packages/gitlab-runner-*.rpm
```

---

## Step 3: Verify the Service

```bash
$ sudo gitlab-runner status
Runtime platform arch=amd64 os=linux pid=52074
gitlab-runner: Service is running

$ sudo gitlab-runner list
Listing configured runners    ConfigFile=/etc/gitlab-runner/config.toml
```

The service starts automatically after installation. If the list is empty, the runner isn't registered yet — that's expected.

---

## Step 4: Register the Runner

To register, you need a **registration token** from your GitLab instance:

1. Go to `http://192.168.8.130/admin/runners`
2. Copy the token under **"Register an instance runner"**
3. Or from any project: **Settings → CI/CD → Runners**

Then register:

```bash
sudo gitlab-runner register \
  --url http://192.168.8.130 \
  --registration-token <your-token> \
  --executor docker \
  --docker-image alpine:latest \
  --description "ops-blog-runner" \
  --tag-list docker,hugo
```

This creates the config at `/etc/gitlab-runner/config.toml`.

### Registration Parameters

| Parameter | Value | Why |
|---|---|---|
| `--url` | `http://192.168.8.130` | Your GitLab server |
| `--executor` | `docker` | Most flexible for CI |
| `--docker-image` | `alpine:latest` | Default image for jobs |
| `--tag-list` | `docker,hugo` | Tags help route specific jobs to this runner |

---

## Step 5: Verify Registration

```bash
$ sudo gitlab-runner list
Listing configured runners    ConfigFile=/etc/gitlab-runner/config.toml
ops-blog-runner    Executor=docker Token=abc123 URL=http://192.168.8.130
```

In the GitLab UI, the runner will appear under **Admin → Runners** with a green dot (online).

---

## 📊 Summary

| Step | Tool | Size | Time |
|---|---|---|---|
| Add repo | manual repo file | — | 5s |
| Download | Tsinghua mirror | 544 MB | ~1 min |
| Install | `dnf` / `rpm` | — | 20s |
| Register | `gitlab-runner register` | — | 10s |
| **Total** | | | **~2 min** |

## 💡 Lessons Learned

1. **🔗 Tsinghua path differs from official** — Runner uses `el9-x86_64` (not `el/9/x86_64`). Check the directory listing if unsure.
2. **💾 Helper images are huge** — 515 MB for pre-built helper Docker images. This is a one-time download.
3. **📝 Registration needs the web UI** — The registration token is encrypted in the database and can't be extracted easily from the CLI. Get it from the admin page.
4. **🐳 Docker executor is the default choice** — It provides clean isolation per job and supports custom images. Perfect for building Docker images and running multi-language pipelines.

---

*The CI/CD foundation is laid. Time to write some `.gitlab-ci.yml` files.* 🚀
