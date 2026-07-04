+++
date = '2026-07-04T16:04:00+08:00'
draft = false
title = '📋 Jenkins Installation on CentOS Stream 9 (Native Package)'
+++

## 📋 Overview

Jenkins is a self-contained CI/CD server. This post covers installing Jenkins LTS on CentOS Stream 9 using the native RPM — along with the Java 21 dependency that tripped me up.

**Server:** `192.168.8.130` — same box as GitLab

---

## Step 1: Find a Working Mirror

The official Jenkins RPM repo redirects to `https://pkg.jenkins.io/rpm-stable/`. Tsinghua mirror (TUNA) doesn't sync Jenkins RPMs, but **USTC** (`mirrors.ustc.edu.cn`) does:

```bash
$ curl -sL "https://mirrors.ustc.edu.cn/jenkins/rpm-stable/" | grep -oP "jenkins-[^\"]*\.rpm"
jenkins-2.541.1-1.noarch.rpm
jenkins-2.541.2-1.noarch.rpm
jenkins-2.541.3-1.noarch.rpm
jenkins-2.555.1-1.noarch.rpm
jenkins-2.555.2-1.noarch.rpm
jenkins-2.555.3-1.noarch.rpm
```

Latest LTS at the time: **2.555.3**. Downloaded in seconds at ~15 MB/s. 🇨🇳

---

## Step 2: Download and Install

```bash
curl -sL -o /tmp/jenkins.rpm \
  "https://mirrors.ustc.edu.cn/jenkins/rpm-stable/jenkins-2.555.3-1.noarch.rpm"

sudo rpm -ivh /tmp/jenkins.rpm
```

The RPM is 96 MB — mostly the Jenkins WAR file and bundled plugins.

---

## 🔥 Java 21 Required — But Distro Has Java 17

First start attempt failed:

```
$ sudo systemctl start jenkins
Job for jenkins.service failed

$ sudo journalctl -xeu jenkins.service
Running with Java 17 from /usr/lib/jvm/java-17-openjdk-...,
which is older than the minimum required version (Java 21).
```

Jenkins 2.555 dropped Java 17 support. CentOS Stream 9 ships Java 17 by default.

**Fix:** Install Java 21 from the same distro repo:

```bash
$ sudo dnf install -y java-21-openjdk-headless
Installed: java-21-openjdk-headless-21.0.11.0.10-2.el9.x86_64
```

No external repo needed — AppStream has Java 21 available.

Then configure Jenkins to use it:

```bash
sudo tee /etc/sysconfig/jenkins > /dev/null <<"EOF"
JENKINS_JAVA_CMD="/usr/lib/jvm/java-21-openjdk-21.0.11.0.10-2.el9.x86_64/bin/java"
JENKINS_HOME="/var/lib/jenkins"
JENKINS_USER="jenkins"
JENKINS_PORT="8080"
JENKINS_LISTEN_ADDRESS="0.0.0.0"
EOF
```

---

## Step 3: Start Jenkins

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now jenkins
```

Status:

```
● jenkins.service - Jenkins Continuous Integration Server
   Loaded: loaded (/usr/lib/systemd/system/jenkins.service; enabled)
   Active: active (running) since 2026-07-04 16:08:53 HKT
 Main PID: 41197 (java)
   Memory: 534.0M (peak: 536.4M)
   Tasks: 50
```

~530 MB at startup — lighter than GitLab. ✅

---

## Step 4: Get the Unlock Key

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
0eca4bda66dc44e49bc38912175fafb0
```

---

## Step 5: Access the Web UI

```
http://192.168.8.130:8080
```

1. Paste the initial admin password
2. Choose **Install suggested plugins** (~5 min)
3. Create your admin user
4. Set the Jenkins URL (keep `http://192.168.8.130:8080`)

---

## 📊 Summary

| Step | Source | Size | Time |
|---|---|---|---|
| Download RPM | USTC mirror | 96 MB | ~10s |
| Install Java 21 | CentOS AppStream | 40 MB | ~20s |
| Install Jenkins | `rpm -ivh` | — | 5s |
| Configure + start | Java 21 path fix | — | 10s |
| **Total** | | | **~1 min** |

## 💡 Lessons Learned

1. **☕ Jenkins 2.555 needs Java 21** — CentOS Stream 9 ships Java 17 by default, but `java-21-openjdk-headless` is available from the same AppStream repo. No external Java install needed.
2. **🔍 Tsinghua doesn't sync Jenkins RPMs** — Use **USTC** (`mirrors.ustc.edu.cn`) instead. TUNA only syncs Jenkins WAR and plugin files, not the RPM repo.
3. **🐌 Jenkins is RAM-light** — 530 MB at startup vs GitLab's 2+ GB. Runs comfortably alongside GitLab on a 3.5 GB server.
4. **🔧 The Jenkins config file** — `/etc/sysconfig/jenkins` controls ports, Java path, and user. Useful for customizing the install.

---

*Two CI/CD servers on one box: GitLab for repos + pipelines, Jenkins for jobs. Next up: wiring them together.* 🔗
