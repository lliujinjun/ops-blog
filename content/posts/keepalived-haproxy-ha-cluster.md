---
title: "🔁 Building a Keepalived + HAProxy HA Cluster on CentOS Stream 9"
date: 2026-07-05T13:42:00+08:00
draft: false
description: "Step-by-step guide to setting up a high-availability load balancer cluster with Keepalived and HAProxy across 3 CentOS Stream 9 VMs."
tags: ["keepalived", "haproxy", "ha", "loadbalancing", "centos"]
categories: ["infrastructure"]
---

High availability doesn't have to be complicated. Keepalived + HAProxy is the classic combo — simple config, reliable failover, and no fancy dependencies.

<!--more-->

---

## What We're Building

A 3-node HA load balancer cluster with a floating virtual IP (VIP):

```
        192.168.211.100  ← VIP
               │
    ┌──────────┼──────────┐
    │          │          │
┌───┴────┐ ┌───┴────┐ ┌───┴────┐
│ lb-1   │ │ lb-2   │ │ lb-3   │
│:80     │ │:80     │ │:80     │  ← HAProxy
│:8443   │ │:8443   │ │:8443   │
└────────┘ └────────┘ └────────┘
```

| Node | IP | Keepalived Role | Priority |
|---|---|---|---|
| **lb-1** | 192.168.211.134 | MASTER | 150 |
| **lb-2** | 192.168.211.135 | BACKUP | 120 |
| **lb-3** | 192.168.211.136 | BACKUP | 110 |
| **VIP** | **192.168.211.100** | Floating | — |

How it works:

- **Keepalived** runs VRRP — the 3 nodes talk to each other, and whichever has the highest priority holds the VIP
- **HAProxy** runs on every node — when a client hits the VIP, it reaches the local HAProxy
- **Health check** — Keepalived monitors HAProxy. If HAProxy dies, Keepalived lowers that node's priority, and the VIP moves
- **Load balancing** — HAProxy forwards traffic to your real backends (web servers, APIs, etc.)

---

## Prerequisites

- 3 CentOS Stream 9 VMs
- Root/sudo access on all
- All VMs on the same subnet

---

## Step 1: Set Hostnames and Hosts

```bash
# On each node
sudo hostnamectl set-hostname lb-1   # or lb-2, lb-3

# Add all to /etc/hosts (run on every node)
echo '192.168.211.134 lb-1'  | sudo tee -a /etc/hosts
echo '192.168.211.135 lb-2'  | sudo tee -a /etc/hosts
echo '192.168.211.136 lb-3'  | sudo tee -a /etc/hosts
echo '192.168.211.100 lb-vip' | sudo tee -a /etc/hosts
```

---

## Step 2: Install Keepalived + HAProxy

```bash
# On all 3 nodes
sudo dnf install -y keepalived haproxy
```

---

## Step 3: Configure HAProxy

Same config on all 3 nodes at `/etc/haproxy/haproxy.cfg`:

```bash
sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo tee /etc/haproxy/haproxy.cfg > /dev/null << 'EOF'
global
    log /dev/local0
    maxconn 4096
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    retries 3
    timeout connect 5s
    timeout client  30s
    timeout server  30s

# Example: Web backend
frontend web_frontend
    bind *:80
    mode http
    default_backend web_servers

backend web_servers
    mode http
    balance roundrobin
    option httpchk GET /
    server web-1 192.168.211.134:8080 check fall 3 rise 2
    server web-2 192.168.211.135:8080 check fall 3 rise 2
    server web-3 192.168.211.136:8080 check fall 3 rise 2

# Example: API backend
frontend api_frontend
    bind *:8443
    mode tcp
    default_backend api_servers

backend api_servers
    mode tcp
    balance roundrobin
    option tcp-check
    server api-1 192.168.211.134:9443 check fall 3 rise 2
    server api-2 192.168.211.135:9443 check fall 3 rise 2
    server api-3 192.168.211.136:9443 check fall 3 rise 2
EOF
```

Replace the backend servers with your actual services.

---

## Step 4: Configure Keepalived

### lb-1 (MASTER, priority 150)

`/etc/keepalived/keepalived.conf`:

```conf
global_defs {
    router_id LB-1
    enable_script_security
}

vrrp_script check_haproxy {
    script "/usr/bin/pgrep haproxy"
    interval 2
    weight -50
    fall 2
    rise 2
    user root
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass k8sha
    }
    virtual_ipaddress {
        192.168.211.100/24
    }
    track_script {
        check_haproxy
    }
}
```

### lb-2 (BACKUP, priority 120)

Same config, but change:

```
router_id LB-2
state BACKUP
priority 120
```

### lb-3 (BACKUP, priority 110)

Same config, but change:

```
router_id LB-3
state BACKUP
priority 110
```

---

## Step 5: Open Firewall and Start Services

```bash
# On all 3 nodes
sudo firewall-cmd --permanent --add-port={80,443,8443}/tcp
sudo firewall-cmd --permanent --add-protocol=vrrp
sudo firewall-cmd --reload

sudo systemctl enable --now haproxy
sudo systemctl enable --now keepalived
```

---

## Step 6: Verify

```bash
# Check VIP location
ip addr show ens33 | grep 192.168.211.100

# Should show on lb-1 (MASTER)
```

Expected output on lb-1:

```
inet 192.168.211.100/24 scope global secondary ens33
```

lb-2 and lb-3 should NOT show the VIP.

---

## Step 7: Test Failover

```bash
# Kill HAProxy on the MASTER
sudo systemctl stop haproxy

# Wait ~5 seconds, then check who has the VIP now
ip addr show ens33 | grep 192.168.211.100
```

The VIP should move to lb-2 or lb-3 automatically. Restore HAProxy and the VIP returns:

```bash
sudo systemctl start haproxy
```

---

## How It Works Under the Hood

**Keepalived's VRRP** sends multicast advertisements every second. Each node announces its priority:

- lb-1: priority 150 (with HAProxy running)
- lb-2: priority 120
- lb-3: priority 110

The node with the highest priority becomes MASTER and holds the VIP. When a node fails or loses HAProxy:

1. The health check script (`pgrep haproxy`) returns non-zero
2. Keepalived subtracts the weight (-50) from the priority
3. lb-1's effective priority drops from 150 to 100
4. lb-2 (priority 120) sees that it's now higher and takes over the VIP
5. Clients experience a brief pause (~10 seconds), then traffic flows again

**HAProxy** provides the actual load balancing. It maintains its own health checks against the real backend servers and only routes traffic to healthy ones. The `balance roundrobin` directive distributes requests evenly.

---

## What's Next?

- **Add real backends** — point HAProxy at your actual services (web apps, APIs, databases)
- **Monitor the VIP** — add a simple health check from a monitoring tool like Prometheus
- **Add more frontends** — HAProxy can handle multiple ports and protocols
- **Customize backends** — weighted distribution, session stickiness, backup servers

---

## Cleanup

```bash
sudo systemctl stop keepalived haproxy
sudo dnf remove -y keepalived haproxy
sudo rm -f /etc/keepalived/keepalived.conf /etc/haproxy/haproxy.cfg
```

---

## Key Takeaways

- Keepalived + HAProxy is the simplest battle-tested HA pattern
- VRRP handles VIP failover in seconds
- HAProxy handles intelligent load balancing with health checks
- 3 nodes give you quorum and survive any single node failure
- The config is portable — the same setup works on Ubuntu, Debian, or RHEL

---

*Built on CentOS Stream 9 with Keepalived 2.2.8 and HAProxy 2.8.*
