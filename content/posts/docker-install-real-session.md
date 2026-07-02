+++
date = '2026-07-02T16:46:00+08:00'
draft = false
title = '🐳 Docker Install Diary: A Real Session on CentOS 8'
+++

This is a companion to the [Docker Engine Installation]({{< relref "/posts/docker-install.md" >}}) guide — what actually happened when I ran it on a real CentOS 8 server at `192.168.8.128`.

The blog post assumes **smooth sailing**. Real life? Not so much. 🌊

---

## 1. 🧹 Clean Slate — Smooth

```bash
sudo dnf remove -y docker docker-client docker-client-latest \
  docker-common docker-latest docker-latest-logrotate \
  docker-logrotate docker-engine podman runc
```

Output: `Nothing to do. Complete!` — clean system, no surprises. ✅

## 2. 📦 Add the Repo — Smooth

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

`dnf-plugins-core` was already installed. Repo added without issues. ✅

## 3. 🚚 Install Docker — Smooth, with a Warning ⚠️

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

103 MB downloaded, GPG key imported. But there was a warning:

```
warning: could not load docker-af-alg-deny.cil SELinux policy; AF_ALG SELinux denial is not active
```

**What this means:** This is harmless on CentOS 8. It's an SELinux policy module that restricts the `AF_ALG` crypto socket family. Docker works fine without it — just a packaging quirk. 😐

## 4. ▶️ Start Docker — Smooth

```bash
sudo systemctl enable --now docker
```

Output: `active (running)`. Docker v29.6.1 up in seconds. ⚡

## 5. 👤 Add User to Docker Group — Smooth, with a Catch 🪤

```bash
sudo usermod -aG docker jellyfish
```

**The catch:** `$USER` doesn't work over SSH since `sudo` resets it. I used the explicit username.

And the `docker` group won't take effect until the user **logs out and back in**. Even `newgrp docker` failed over SSH because it spawns a subshell that can't reach the daemon socket.

**Lesson for the blog:** The blog says "log out and back in" — this is more than a suggestion. If you're SSH'd in, you need to `exit` and reconnect. 🚪

## 6. 💥 Hello World? ❌ Network Failure

This is where things went off-script:

```bash
sudo docker run hello-world
```

```
Unable to find image 'hello-world:latest' locally
docker: Error response from daemon: failed to resolve reference
"docker.io/library/hello-world:latest": failed to do request:
Head "https://registry-1.docker.io/v2/library/hello-world/manifests/latest":
read tcp 192.168.8.128:42330->198.18.0.15:443: read: connection reset by peer
```

**Diagnosis:** `connection reset by peer` — classic China firewall behavior. Docker Hub is intermittently blocked. 🔥🚧

## 7. 🔍 Mirror Hunt — Three Strikes 🎯

Set up a registry mirror in `/etc/docker/daemon.json`:

### Strike 1: USTC Mirror 🥇

```json
{"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]}
```

Result: `EOF` — unreachable from this server. ❌

### Strike 2: NetEase 163 Mirror 🥈

```json
{"registry-mirrors": ["https://hub-mirror.c.163.com"]}
```

Result: `EOF` — also unreachable. ❌

### Strike 3: Direct Docker Hub Check 🥉

```bash
curl -s -o /dev/null -w "%{http_code}" https://registry-1.docker.io/v2/
```

Result: `401` — **that's a good sign!** A 401 means the server reached Docker Hub just fine; it returned "unauthorized" as expected for an unauthenticated request to `/v2/`.

So the **mirrors were the problem**, not Docker Hub. 🤦

## 8. 🏁 Back to Direct — Success 🎉

```bash
sudo rm -f /etc/docker/daemon.json
sudo systemctl restart docker
sudo docker run hello-world
```

And finally:

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

**Lesson:** The initial failure was a transient network glitch, not a permanent block. Trying again (or waiting a few minutes) would have worked. The mirrors were actually worse than going direct. 🙃

---

## 📊 Summary of Real-World Differences

| Blog Guide | Reality |
|---|---|
| Everything works first try | `hello-world` failed — connection reset 💥 |
| Mirrors are a nice option | Both USTC and 163 mirrors returned EOF 🚫 |
| No SELinux warnings | `docker-af-alg-deny.cil` warning appeared ⚠️ |
| `$USER` works in example | Had to use explicit username over SSH 👤 |
| "Log out and back in" | Really means it — `newgrp` doesn't work over SSH 🚪 |

## 💡 Lessons Learned

1. **🔍 Don't blindly trust mirrors** — test them first with `curl`. A bad mirror is worse than no mirror.
2. **⏳ Transient failures happen** — Docker Hub had a hiccup, then worked fine. Always retry before diving into config changes.
3. **🔗 SSH adds friction** — group membership changes, sudo behavior, and subshells all behave differently over SSH than on a local terminal.
4. **📖 The blog is a recipe; reality is cooking** — the guide gets you 80% there. The last 20% is debugging. 🔧
