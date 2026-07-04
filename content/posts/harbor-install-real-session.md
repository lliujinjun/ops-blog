+++
date = '2026-07-04T10:49:00+08:00'
draft = false
title = '📦 Harbor Install Diary: A Real Session on CentOS 8'
+++

This is a companion to the [Harbor Installation Guide]({{< relref "/posts/harbor-installation.md" >}}) — what actually happened when I installed it on a real CentOS 8 server at `192.168.8.128`.

The guide makes it look clean. Reality? Less so. Let's walk through each step and what went wrong. 🔧

---

## 0. Starting Point

The server already had Docker 26.1.3 and Docker Compose v2.27.0 running from a [previous session]({{< relref "/posts/docker-install-real-session.md" >}}). So I skipped directly to Harbor.

```bash
$ ssh 192.168.8.128
$ docker --version
Docker version 26.1.3, build b72abbb
$ docker compose version
Docker Compose version v2.27.0
```

All good so far. ✅

---

## 1. 📥 Downloading Harbor — Smooth

```bash
$ cd /tmp
$ curl -sL https://github.com/goharbor/harbor/releases/latest/download/harbor-offline-installer-v2.15.2.tgz -o harbor-offline-installer.tgz
$ tar xzf harbor-offline-installer.tgz
$ sudo mv harbor /opt/harbor
```

Harbor v2.15.2 — latest at the time. The offline tarball is ~700 MB. Download took a couple minutes but finished clean. ✅

## 2. ⚙️ Config — Smooth Until It Wasn't

```bash
$ cd /opt/harbor
$ cp harbor.yml.tmpl harbor.yml
```

I changed three things:

1. **hostname** → `192.168.8.128` (no domain yet)
2. **HTTPS** → disabled (no cert)
3. **admin password** → kept default for now

### 🔥 The HTTPS Surgery

The guide says "comment out the HTTPS section." In YAML that means `# https:` and all its children. But the template has mixed indentation and inline comments, so a simple `sed` left me with this mess:

```yaml
# https:
  # https port for harbor, default is 443
#   port: 443
  # The path of cert and key files for nginx
#   certificate: /your/certificate/path
#   private_key: /your/private/key/path
  # enable strong ssl ciphers (default: false)
```

Some lines got commented, some didn't. This kind of **semi-broken YAML is a ticking time bomb** — it might parse fine today and fail on the next `./install.sh`.

**Fix:** I used a Python script to do it properly — check each line, and if it's inside the `https:` block, prepend `# ` to every uncommented line:

```python
with open("harbor.yml", "r") as f:
    lines = f.read().split("\n")
new_lines = []
skip_https = False
for line in lines:
    if line.strip() == "https:":
        skip_https = True
        new_lines.append("# https:")
        continue
    if skip_https:
        if line and not line[0].isspace() and not line.startswith("#") and line.strip() != "":
            skip_https = False
            new_lines.append(line)
            continue
        if line.strip() and not line.strip().startswith("#") and not line.strip().startswith("  #"):
            new_lines.append("# " + line)
        else:
            new_lines.append(line)
        continue
    new_lines.append(line)
```

**Lesson:** Never trust `sed` to comment out multi-line YAML blocks. Write a real parser or use `yq`. 📝

## 3. 🔌 Running the Installer — Smooth(ish)

```bash
$ cd /opt/harbor
$ sudo bash install.sh
```

### Step 0-1: Docker & Compose checks ✅

```
Note: docker version: 26.1.3
Note: Docker Compose version v2.27.0
```

### Step 2: Loading images 📦

```
Loaded image: goharbor/harbor-exporter:v2.15.2
Loaded image: goharbor/harbor-portal:v2.15.2
Loaded image: goharbor/harbor-db:v2.15.2
Loaded image: goharbor/harbor-registryctl:v2.15.2
Loaded image: goharbor/nginx-photon:v2.15.2
Loaded image: goharbor/registry-photon:v2.15.2
Loaded image: goharbor/harbor-core:v2.15.2
Loaded image: goharbor/harbor-jobservice:v2.15.2
Loaded image: goharbor/valkey-photon:v2.15.2
Loaded image: goharbor/prepare:v2.15.2
Loaded image: goharbor/harbor-log:v2.15.2
Loaded image: goharbor/trivy-adapter-photon:v2.15.2
```

All 12 images loaded from the offline tarball — no network needed. This is why offline installer is worth the 700 MB. 🏋️

### Step 3-4: Config generation ⚙️

```
WARNING:root:WARNING: HTTP protocol is insecure.
Harbor will deprecate http protocol in the future.
Please make sure to upgrade to https
```

This fires for any HTTP-only setup. Not a blocker, but Harbor v2.15.2 is the first version to push hard on this deprecation. Future versions may warn louder or eventually refuse. 🔔

### Step 5: Starting containers 🚀

All 9 containers spun up in order:

1. `harbor-log` — centralized logging
2. `harbor-db` — PostgreSQL
3. `redis` — cache layer
4. `registryctl` — registry controller
5. `harbor-portal` — web UI
6. `registry` — image storage
7. `harbor-core` — API server
8. `harbor-jobservice` — async jobs
9. `nginx` — reverse proxy

```
✔ ----Harbor has been installed and started successfully.----
```

## 4. ✅ Verification

```bash
$ curl -s http://192.168.8.128/api/v2.0/ping
{"ping":"pong"}
```

```bash
$ curl -s -o /dev/null -w "%{http_code}" http://192.168.8.128/
200
```

API responds, UI serves. Let's test a login:

```bash
$ docker login 192.168.8.128 --username admin
Password: Harbor12345
WARNING! Your password will be stored unencrypted...
Login Succeeded
```

Push a test image:

```bash
$ docker pull alpine:3.19
$ docker tag alpine:3.19 192.168.8.128/library/alpine:3.19
$ docker push 192.168.8.128/library/alpine:3.19
The push refers to repository [192.168.8.128/library/alpine]
...
latest: digest: sha256:abc123... size: 528
```

**Push succeeded. Harbor is fully operational.** 🎉

> ⚠️ **Note on HTTP:** Because I'm using HTTP without TLS, Docker requires the registry to be listed as insecure:
> ```json
> { "insecure-registries": ["192.168.8.128"] }
> ```
> Add this to `/etc/docker/daemon.json` and restart Docker. Since both client and server are on the same LAN, this is acceptable for a lab environment.

---

## 📊 Summary of Real-World Differences

| Guide Clean | Reality |
|---|---|
| `sed` to comment HTTPS block | Semi-broken YAML, had to write Python 🐍 |
| All YAML fields just work | Indentation and inline comments break naive parsing 💥 |
| Install script passes cleanly | HTTP deprecation warning logged 🔊 |
| HTTPS recommended but optional | Harbor is actively deprecating HTTP 🚩 |
| Config changes take effect immediately | Containers need full restart after config change 🔄 |

---

## 💡 Lessons Learned

1. **🐍 Python > sed for YAML surgery** — `sed` is great for single-line replacements. For multi-line YAML blocks, use a proper parser or at least a Python script that understands indentation context.
2. **📦 Offline installer is worth it** — 700 MB download once saves downloading 12 images individually over potentially flaky network connections.
3. **🔔 Read the warnings** — Harbor's HTTP deprecation message isn't noise. Plan for HTTPS before production.
4. **🔄 Config changes mean full restart** — `docker compose down && sudo ./install.sh`, not just `docker compose restart`.
5. **⚡ Docker Compose v2 is fast** — Harbor 2.15.2 went from `install.sh` to all containers healthy in under 30 seconds.

**Final state:** Harbor v2.15.2, HTTP on port 80, `admin` / `Harbor12345`, ready for projects, users, and replication rules. 🏁
