---
title: "🗄️ Ceph RADOS Storage Cluster Setup on CentOS Stream 9"
date: 2026-07-05T19:56:00+08:00
draft: false
description: "Build a bare-metal Ceph RADOS storage cluster on CentOS Stream 9 — just the storage layer, no filesystem, no block devices, no S3 gateway."
tags: ["ceph", "rados", "storage", "centos", "homelab"]
categories: ["storage"]
---

Just the RADOS layer — the foundation of Ceph. No CephFS, no RBD, no RadosGW. Pure distributed object storage.

<!--more-->

---

## What is RADOS?

RADOS (Reliable Autonomic Distributed Object Store) is the core of Ceph. Everything else — CephFS, RBD block devices, RadosGW S3 — is just a client sitting on top.

A RADOS cluster has three components:

| Component | Role |
|---|---|
| **Monitor (MON)** | Maintains the cluster map. Uses Paxos for consensus. Needs quorum. |
| **Manager (MGR)** | Handles metrics, rebalancing, dashboard, orchestration. Not in the data path. |
| **OSD** | The actual storage daemon. One per disk. Handles replication, recovery, rebalancing. |

Data placement is driven by **CRUSH** (Controlled Replication Under Scalable Hashing) — a mathematical function that maps objects to OSDs without a central index. No metadata server, no lookup bottleneck.

---

## What We're Building

A single-node Ceph Squid cluster with 3 OSDs using loopback files:

```
┌────────────────────────────────────┐
│         ceph-node1                 │
│   CentOS Stream 9                  │
│                                    │
│  ┌─────┐  ┌─────┐  ┌───────────┐  │
│  │ MON │  │ MGR │  │ OSD.0     │  │
│  │     │  │     │  │ OSD.1     │  │
│  │     │  │     │  │ OSD.2     │  │
│  └─────┘  └─────┘  └───────────┘  │
│                     3 x ~10GB     │
└────────────────────────────────────┘
```

Since this is a single node, replication is set to `size = 1`. In production with 3+ nodes, you'd raise this to 3.

---

## Prerequisites

- CentOS Stream 9 VM
- 3.5GB+ RAM
- 30GB+ free disk space
- Root/sudo access

---

## Step 1: Set Hostname

```bash
sudo hostnamectl set-hostname ceph-node1
echo '192.168.211.133 ceph-node1' | sudo tee -a /etc/hosts
```

Replace the IP with your VM's actual address.

---

## Step 2: Install Ceph Packages

Ceph Squid (19.x) is the latest stable release series:

```bash
sudo dnf install -y epel-release
sudo dnf config-manager --add-repo \
  https://download.ceph.com/rpm-squid/el9/x86_64/
sudo dnf config-manager --add-repo \
  https://download.ceph.com/rpm-squid/el9/noarch/

sudo dnf install -y --nogpgcheck \
  ceph-mon ceph-mgr ceph-osd ceph-common ceph-selinux
```

> The `noarch` repo is required — `ceph-mgr` depends on `ceph-mgr-modules-core` which is a noarch package. Without it, installation fails with a dependency error.

Verify:

```bash
ceph --version
# ceph version 19.2.4 (squid)
```

---

## Step 3: Create the Configuration

```ini
# /etc/ceph/ceph.conf
[global]
fsid = a1b2c3d4-e5f6-7890-abcd-ef1234567890
mon_host = 192.168.211.133
public_network = 192.168.211.0/24
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
osd_pool_default_size = 1
osd_pool_default_min_size = 1
mon_max_pg_per_osd = 1024

[mon]
keyring = /var/lib/ceph/mon/ceph-ceph-node1/keyring

[mgr]
keyring = /var/lib/ceph/mgr/ceph-ceph-node1/keyring

[client.admin]
keyring = /etc/ceph/ceph.client.admin.keyring

[client.bootstrap-osd]
keyring = /var/lib/ceph/bootstrap-osd/ceph.keyring
```

Generate your own `fsid` with the `uuidgen` command.

---

## Step 4: Generate Keys

```bash
# Monitor key
sudo ceph-authtool --create-keyring /tmp/ceph.mon.keyring \
  --gen-key -n mon. --cap mon "allow *"

# Admin key (full access)
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring \
  --gen-key -n client.admin \
  --cap mon "allow *" --cap osd "allow *" \
  --cap mds "allow *" --cap mgr "allow *"

# Merge admin key into mon keyring
sudo ceph-authtool /tmp/ceph.mon.keyring \
  --import-keyring /etc/ceph/ceph.client.admin.keyring

# Bootstrap keys for OSDs, RBD, and RadosGW
for role in osd rbd rgw; do
  sudo mkdir -p /var/lib/ceph/bootstrap-$role
  sudo ceph-authtool --create-keyring \
    /var/lib/ceph/bootstrap-$role/ceph.keyring --gen-key \
    -n client.bootstrap-$role \
    --cap mon "allow profile bootstrap-$role"
done
```

---

## Step 5: Bootstrap the Monitor

```bash
# Create monitor map
sudo monmaptool --create --add ceph-node1 192.168.211.133 \
  --fsid a1b2c3d4-e5f6-7890-abcd-ef1234567890 /tmp/monmap

# Initialize monitor data directory
sudo mkdir -p /var/lib/ceph/mon/ceph-ceph-node1
sudo ceph-mon --mkfs -i ceph-node1 \
  --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
sudo rm /tmp/ceph.mon.keyring /tmp/monmap
sudo chown -R ceph:ceph /var/lib/ceph /etc/ceph

# Start the monitor
sudo systemctl enable --now ceph-mon@ceph-node1
```

The first start may fail if the monitor keyring wasn't properly created in the data directory. Fix it:

```bash
# If the keyring is missing from the mon data directory
sudo ceph-authtool --create-keyring \
  /var/lib/ceph/mon/ceph-ceph-node1/keyring \
  --gen-key -n mon. --cap mon "allow *"
sudo chown ceph:ceph /var/lib/ceph/mon/ceph-ceph-node1/keyring
sudo systemctl restart ceph-mon@ceph-node1
```

Verify:

```bash
sudo ceph -s
```

```
cluster:
    id:     a1b2c3d4-...
    health: HEALTH_OK

services:
    mon: 1 daemons, quorum ceph-node1
```

---

## Step 6: Start the Manager

```bash
sudo mkdir -p /var/lib/ceph/mgr/ceph-ceph-node1
sudo ceph auth get-or-create mgr.ceph-node1 \
  mon "allow profile mgr" osd "allow *" mds "allow *" \
  -o /var/lib/ceph/mgr/ceph-ceph-node1/keyring
sudo chown -R ceph:ceph /var/lib/ceph/mgr

sudo systemctl enable --now ceph-mgr@ceph-node1
```

Confirm it's active:

```bash
sudo ceph -s
# Look for: mgr: ceph-node1(active)
```

---

## Step 7: Create OSDs

Since there are no spare block devices on a typical VM, we create loopback files backed by LVM.

```bash
# Install LVM
sudo dnf install -y lvm2

# Create 3 x 10GB loopback files
for i in 0 1 2; do
  sudo dd if=/dev/zero of=/home/ceph-osd-$i.img bs=1M count=10240
  sudo losetup /dev/loop$i /home/ceph-osd-$i.img
done

# Partition and set up LVM
for i in 0 1 2; do
  sudo parted -s /dev/loop$i mklabel gpt
  sudo parted -s /dev/loop$i mkpart primary 0% 100%
  sudo pvcreate /dev/loop${i}p1
  sudo vgcreate ceph-vg-$i /dev/loop${i}p1
  sudo lvcreate -n ceph-lv-$i -l 100%FREE ceph-vg-$i
done
```

Now create the OSDs:

```bash
# Import the bootstrap key into the cluster
sudo ceph auth import -i /var/lib/ceph/bootstrap-osd/ceph.keyring

# Let ceph-volume handle the rest
for i in 0 1 2; do
  sudo ceph-volume lvm create --data ceph-vg-$i/ceph-lv-$i
done
```

The OSD daemon expects its data directory at `/var/lib/ceph/osd/$id` but ceph-volume names them `ceph-$id`. Fix with a symlink:

```bash
for i in 0 1 2; do
  if sudo test -d /var/lib/ceph/osd/ceph-$i; then
    sudo ln -s /var/lib/ceph/osd/ceph-$i /var/lib/ceph/osd/$i
    sudo systemctl start ceph-osd@$i
  fi
done
```

---

## Step 8: Verify the Cluster

```bash
sudo ceph -s
```

```
cluster:
    id:     a1b2c3d4-e5f6-7890-abcd-ef1234567890
    health: HEALTH_WARN
            mon is allowing insecure global_id reclaim
            1 pool(s) have no replicas configured

services:
    mon: 1 daemons, quorum ceph-node1
    mgr: ceph-node1(active)
    osd: 3 osds: 3 up, 3 in

data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 449 KiB
    usage:   79 MiB used, 30 GiB / 30 GiB avail
    pgs:     1 active+clean
```

```bash
sudo ceph osd tree
```

```
ID  CLASS  WEIGHT   TYPE NAME       STATUS
-1         0.02939  root default
-3         0.02939      host ceph-node1
 0    hdd  0.00980          osd.0  up  1.00000
 1    hdd  0.00980          osd.1  up  1.00000
 2    hdd  0.00980          osd.2  up  1.00000
```

The `HEALTH_WARN` is expected — we set `osd_pool_default_size = 1` because a single node can't replicate. In production with 3+ nodes, set it to 3 and the warning disappears.

---

## What You Can Do With Just RADOS

Without CephFS, RBD, or RadosGW, you still have a fully functional distributed object store:

```bash
# Write an object
echo "Hello RADOS" | sudo rados -p .mgr put hello -

# List objects
sudo rados -p .mgr ls

# Read an object
sudo rados -p .mgr get hello -

# Create a pool
sudo ceph osd pool create mypool 32

# Delete an object
sudo rados -p .mgr rm hello
```

---

## Troubleshooting

### Monitor fails to start: missing keyring

```
unable to load initial keyring /var/lib/ceph/mon/ceph-ceph-node1/keyring
```

Create the keyring manually:

```bash
sudo ceph-authtool --create-keyring \
  /var/lib/ceph/mon/ceph-ceph-node1/keyring \
  --gen-key -n mon. --cap mon "allow *"
sudo chown ceph:ceph /var/lib/ceph/mon/ceph-ceph-node1/keyring
```

### Dependency error during install

```
nothing provides ceph-mgr-modules-core = 2:19.2.4-0.el9
```

Add the noarch repo:

```bash
sudo dnf config-manager --add-repo \
  https://download.ceph.com/rpm-squid/el9/noarch/
```

### IP mismatch in ceph.conf

The `public_network` in ceph.conf must match the VM's actual subnet. Check with `ip addr` and update the config:

```bash
ip -4 addr show | grep inet | grep -v 127.0.0.1
# Update ceph.conf if needed, then restart the mon
```

---

## Cleanup

```bash
sudo systemctl stop ceph-{mon,mgr,osd}@ceph-node1
for i in 0 1 2; do
  sudo systemctl stop ceph-osd@$i
done
sudo dnf remove -y ceph-mon ceph-mgr ceph-osd ceph-common
sudo rm -rf /var/lib/ceph /etc/ceph
for i in 0 1 2; do
  sudo losetup -d /dev/loop$i
  sudo rm -f /home/ceph-osd-$i.img
done
```

---

## What's Next?

- **Add CephFS** — create a filesystem on top of RADOS
- **Add RBD** — provision block devices for VMs or containers
- **Add RadosGW** — deploy an S3-compatible object storage gateway
- **Add more nodes** — expand the cluster to 3+ nodes for replication and HA

---

*Built on CentOS Stream 9 with Ceph Squid 19.2.4. Single-node test cluster — not for production.*
