+++
date = '2026-07-04T14:57:00+08:00'
draft = false
title = '🔧 GitLab Omnibus Install on CentOS Stream 9 (With a Tuna Twist)'
+++

## 📋 Overview

This post documents installing GitLab CE 19.1.1 on a CentOS Stream 9 server using the Omnibus package — but with a critical detour through a Chinese mirror to survive the Great Firewall's slow connection to Google Cloud Storage.

**Server:** `192.168.8.130` — CentOS Stream 9, 3.5 GB RAM, x86_64

---

## ⚠️ Resource Warning

GitLab officially recommends **4 GB RAM minimum**. This server has 3.5 GB. It runs, but startup is slow and heavy CI pipelines might struggle. Adding swap or upgrading RAM is recommended for production.

---

## Step 0: Install Dependencies

```bash
sudo dnf install -y curl policycoreutils openssh-server openssh-clients postfix
sudo systemctl enable --now postfix
```

Standard stuff. Postfix is needed for GitLab's email notifications (password resets, pipeline notifications, etc.).

---

## Step 1: Add the GitLab Repository

The official way is the installer script:

```bash
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
```

**But:** On CentOS Stream 9, the script kept getting killed mid-way (timeout issues). The manual repo approach was more reliable:

```bash
sudo tee /etc/yum.repos.d/gitlab-ce.repo > /dev/null <<"EOF"
[gitlab-ce]
name=gitlab-ce
baseurl=https://packages.gitlab.com/gitlab/gitlab-ce/el/9/$basearch
repo_gpgcheck=1
gpgcheck=1
enabled=1
gpgkey=https://packages.gitlab.com/gitlab/gitlab-ce/gpgkey
EOF
```

---

## 🔥 The Great Firewall Problem

The GitLab CE RPM is **942 MB**. Downloading from the official CDN (Google Cloud Storage) was painfully slow:

| Attempt | Source | Speed | Progress |
|---|---|---|---|
| `dnf install` (official) | GCS CDN | ~200 KB/s | Killed at 785 MB |
| `curl` (official redirect) | GCS CDN | ~1.2 MB/s → 200 KB/s | Got to 474 MB |
| **Tsinghua mirror** 🇨🇳 | TUNA | **~15 MB/s** | **942 MB in <1 min** |

The official CDN keeps redirecting to `storage.googleapis.com`, which has notoriously flaky connectivity from mainland China — `connection reset by peer`, erratic speeds, and timeouts.

### Switch to Tsinghua Mirror

```bash
sudo tee /etc/yum.repos.d/gitlab-ce.repo > /dev/null <<"EOF"
[gitlab-ce]
name=gitlab-ce
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el9/
enabled=1
gpgcheck=0
repo_gpgcheck=0
EOF
```

Then:

```bash
sudo EXTERNAL_URL="http://192.168.8.130" dnf install -y gitlab-ce
```

The `EXTERNAL_URL` environment variable tells GitLab's post-install script what URL to configure in nginx. The download flew at ~15 MB/s and completed in under a minute. 🚀

---

## Step 2: Post-Install Configuration

The `dnf install` triggers `gitlab-ctl reconfigure` automatically as a post-install script. This is the heavy part:

- Sets up PostgreSQL (data directory, users, schema)
- Configures Redis
- Generates nginx config
- Runs chef-based provisioning
- Compiles assets

On this 3.5 GB server, reconfigure took about **3-4 minutes**. During this time, `semodule` (SELinux) and `ruby` (chef) processes were running.

```bash
$ sudo gitlab-ctl status
run: gitaly: (pid 47886) 37s; run: log: (pid 47930) 36s
run: gitlab-kas: (pid 48541) 20s; run: log: (pid 48558) 19s
run: logrotate: (pid 47657) 49s; run: log: (pid 47695) 46s
run: postgresql: (pid 48084) 26s; run: log: (pid 48133) 23s
run: redis: (pid 47751) 43s; run: log: (pid 47777) 42s
```

---

## Step 3: Get the Root Password

After installation, GitLab generates a temporary root password:

```bash
$ sudo cat /etc/gitlab/initial_root_password
Password: lSyI3uiFQqsT6Yd8yhOYuvA19H45+InNMIFH4wXp3hA=
```

**Important:** This file is deleted after 24 hours. Save the password or change it immediately from the web UI.

---

## Step 4: Access the Web UI

```
http://192.168.8.130
```

- **Username:** `root`
- **Password:** (from `/etc/gitlab/initial_root_password`)

First thing to do: go to **Admin → Settings → General** and change the password to something you'll remember.

---

## 📊 Summary

| Step | Tool | Time | Note |
|---|---|---|---|
| Install deps | `dnf` | ~20s | postfix, policycoreutils |
| Add repo | manual repo file | ~1s | Official script kept timing out |
| Download RPM | Tsinghua mirror | **<1 min** | 942 MB @ ~15 MB/s |
| Install + reconfigure | `dnf + gitlab-ctl` | ~4 min | Heavy RAM usage during reconfigure |
| **Total** | | **~5 min** | ✅ GitLab 19.1.1 live |

## 💡 Lessons Learned

1. **🇨🇳 Mirror first, official second** — In China, always start with a domestic mirror. The official GitLab CDN (GCS) is nearly unusable from mainland networks. Tsinghua's TUNA mirror saved the day.
2. **🤏 3.5 GB RAM is tight** — GitLab runs but you'll feel it. The reconfigure step was noticeably slow. Consider swap or more RAM.
3. **📜 Initial password is temporary** — `/etc/gitlab/initial_root_password` is deleted after 24h. Change it immediately.
4. **🔧 `EXTERNAL_URL` matters** — Without it, `dnf install` will prompt for the URL interactively. Setting it upfront avoids a failed first reconfigure.
5. **🎯 Runner is next** — GitLab isn't useful without CI. GitLab Runner installation is covered in the next post.

---

*Next up: GitLab Runner — the engine that actually runs your pipelines.* 🏃
