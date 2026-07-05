+++
date = '2026-07-05T11:48:00+08:00'
draft = false
title = '🔍 Elasticsearch & Kibana Install Diary: A Real Session on CentOS Stream 9'
+++

This is a walkthrough of installing Elasticsearch 8.19.18 and Kibana 8.19.18 **natively** (not Docker) on a CentOS Stream 9 VM — the bumps, retries, and all. If you like clean step-by-step guides, read on anyway — this is the real thing, warts included.

---

## 0. Starting Point

A CentOS Stream 9 VM (`remote-vm` alias in SSH config) with:

```
4 vCPUs
3.5 GB RAM
68 GB free disk
```

No Java, no Elastic stack. Clean slate, except it shares the box with an Android SDK/emulator setup from earlier.

---

## 1. Add the Elastic YUM Repo

Elastic provides official RPM repos for 8.x:

```bash
# Import GPG key
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

# Add repo
cat <<'EOF' | sudo tee /etc/yum.repos.d/elastic.repo
[elastic-8.x]
name=Elastic repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```

Straightforward — no issues here.

---

## 2. Install Elasticsearch — The First Attempt

```bash
sudo dnf -y install elasticsearch
```

The package is **648 MB**. The download started at ~11 MB/s.

Then silence. The SSH session timed out mid-download and the process was killed (`SIGKILL`).

```
[Errno 2] No such file or directory:
  '/var/cache/dnf/elastic-8.x-.../packages/elasticsearch-8.19.18-x86_64.rpm'
```

**Lesson:** Large RPM downloads over SSH need either a longer timeout, a keepalive, or `nohup`. I retried with `ServerAliveInterval=60`:

```bash
ssh -o ServerAliveInterval=60 remote-vm "sudo dnf -y install elasticsearch"
```

Second attempt picked up the cached download, but the metadata was stale. After `dnf clean all && dnf makecache`, the package was already marked as installed — the first attempt had actually finished the install before being killed.

```bash
$ rpm -q elasticsearch
elasticsearch-8.19.18-1.x86_64
```

Elasticsearch 8.19.18 *was* installed. ✅

---

## 3. Configure Elasticsearch — Single Node, No Security

The default config comes with **TLS + password auth** enabled. For a dev/test single-node setup, that's unnecessary overhead. Here's what I changed:

**`/etc/elasticsearch/elasticsearch.yml`:**

```yaml
cluster.name: my-cluster
node.name: node-1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
discovery.type: single-node

# Dev — no TLS, no auth
xpack.security.enabled: false
xpack.security.enrollment.enabled: false
xpack.security.http.ssl.enabled: false
xpack.security.transport.ssl.enabled: false
```

Key decisions:
- `discovery.type: single-node` — no cluster bootstrap needed
- `0.0.0.0` binding — reachable from LAN (fine for internal dev)
- Security disabled — this is intentionally *not* production-ready

JVM heap defaulted to 1 GB, which is fine for 3.5 GB RAM:

```bash
sudo grep -E '^-Xm' /etc/elasticsearch/jvm.options
## -Xms4g   (commented out — default 1 GB auto-configured)
```

Started it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch
```

After ~15 seconds:

```json
{
  "name" : "node-1",
  "cluster_name" : "my-cluster",
  "version" : {
    "number" : "8.19.18",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "lucene_version" : "9.12.2"
  },
  "tagline" : "You Know, for Search"
}
```

Elasticsearch is alive. ✅

---

## 4. Install & Configure Kibana

Same repo, same process:

```bash
sudo dnf -y install kibana
```

**387 MB** — downloaded cleanly this time at 5.6 MB/s. Install created the `kibana` user and group automatically.

**First configuration attempt — and failure:**

```yaml
# /etc/kibana/kibana.yml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.ssl.verificationMode: none
xpack.security.enabled: false   # ❌ THIS LINE!
```

Kibana 8.x rejected it:

```
FATAL Error: [config validation of [xpack.security].enabled]:
  definition for this key is missing
```

In ES 8.x, the `xpack.security` config key doesn't exist in Kibana's schema unless the security plugin bundle is loaded. Removing that line fixed it:

```yaml
# /etc/kibana/kibana.yml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
elasticsearch.ssl.verificationMode: none
```

Then:

```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```

After about 15 seconds, port 5601 responded with HTTP 302 (redirect to the Kibana home page).

Both services active. ✅

---

## 5. Final State

```
$ sudo systemctl is-active elasticsearch kibana
active
active
```

| Component | Version | Port | Status |
|-----------|---------|------|--------|
| Elasticsearch | 8.19.18 | 9200 | ✅ running (2.0 GB RSS) |
| Kibana | 8.19.18 | 5601 | ✅ running (659 MB RSS) |

```bash
# Quick sanity
curl -s http://localhost:9200/_cluster/health | python3 -m json.tool
# -> status: "yellow" (expected for single-node — missing replicas)
curl -s -o /dev/null -w '%{http_code}' http://localhost:5601
# -> 302 (redirect to Kibana UI)
```

---

## Lessons Learned

1. **Large RPMs + SSH = risk.** 648 MB package timed out on the first attempt. Use `ServerAliveInterval` or `screen`/`tmux` for long-running remote installs.

2. **ES 8.x default config is security-hardened.** That's great for production, but for a local dev VM you'll want to strip TLS and auth to reduce friction.

3. **Kibana 8.x doesn't accept `xpack.security.enabled`.** Unlike ES, Kibana's config schema only validates keys that have registered plugin definitions. Remove, don't set false.

4. **Resource usage is real.** After boot: ES ~2 GB, Kibana ~660 MB. That's 2.6/3.5 GB on a small VM. Plan accordingly.

5. **The RPM install path is simple once you get past the download.** No JDK to install manually — ES bundles its own JDK (in `/usr/share/elasticsearch/jdk/`).
