---
title: "📦 Building a Ceph Storage Cluster on CentOS Stream 9"
date: 2026-07-05T13:02:45+08:00
draft: false
description: "Step-by-step guide to setting up a Ceph Squid 19.2.4 cluster on a single CentOS Stream 9 VM for testing and learning."
tags: ["ceph", "storage", "centos", "homelab", "cephfs"]
categories: ["storage"]
---

Ceph Squid 19.2.4 on a single-node test cluster.

<!--more-->

---

## What We're Building

A fully working Ceph cluster with:

- 1 Monitor — the cluster brain
- 1 Manager — metrics and orchestration
- 1 Metadata Server — powers CephFS
- 3 OSDs — ~10GB each, using loopback files
- A mountable Ceph filesystem

All on one CentOS Stream 9 VM. Perfect for learning, dev/test, or a home lab.

---

## Prerequisites

- CentOS Stream 9 VM
- 4GB+ RAM
- Root/sudo access
- systemd running

---

## Step 1: Install Ceph Packages

Add the repository and install only what we need:

```bash
sudo dnf install -y epel-release
sudo dnf config-manager --add-repo \
  https://download.ceph.com/rpm-squid/el9/x86_64/
sudo dnf install -y --nogpgcheck \
  ceph-mon ceph-mgr ceph-osd ceph-mds \
  ceph-common ceph-radosgw ceph-selinux
```

Verify:

```bash
ceph --version
# ceph version 19.2.4 (squid)
```

---

## Step 2: Set the Hostname

Ceph needs a proper hostname. Set it and add it to `/etc/hosts`:

```bash
sudo hostnamectl set-hostname ceph-node1
echo '192.168.211.133 ceph-node1' | sudo tee -a /etc/hosts
```

Replace `192.168.211.133` with your VM's actual IP.

---

## Step 3: Configure the Cluster

Create `/etc/ceph/ceph.conf`:

```ini
[global]
fsid = a1b2c3d4-e5f6-7890-abcd-ef1234567890
mon_host = 192.168.211.133
public_network = 192.168.211.0/24
cluster_network = 192.168.211.0/24
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

Generate your own `fsid` with `uuidgen`.

---

## Step 4: Bootstrap the Monitor

The monitor is the cluster's brain — it maintains the cluster map.

**Generate keys:**

```bash
sudo ceph-authtool --create-keyring /tmp/ceph.mon.keyring \
  --gen-key -n mon. --cap mon "allow *"

sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring \
  --gen-key -n client.admin \
  --cap mon "allow *" --cap osd "allow *" \
  --cap mds "allow *" --cap mgr "allow *"

sudo ceph-authtool /tmp/ceph.mon.keyring \
  --import-keyring /etc/ceph/ceph.client.admin.keyring
```

**Create bootstrap keys:**

```bash
for role in osd rbd rgw; do
  sudo mkdir -p /var/lib/ceph/bootstrap-$role
  sudo ceph-authtool --create-keyring \
    /var/lib/ceph/bootstrap-$role/ceph.keyring --gen-key \
    -n client.bootstrap-$role \
    --cap mon "allow profile bootstrap-$role"
done
```

**Initialize and start:**

```bash
sudo monmaptool --create --add ceph-node1 192.168.211.133 \
  --fsid a1b2c3d4-e5f6-7890-abcd-ef1234567890 /tmp/monmap

sudo mkdir -p /var/lib/ceph/mon/ceph-ceph-node1
sudo ceph-mon --mkfs -i ceph-node1 \
  --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring
sudo rm /tmp/ceph.mon.keyring /tmp/monmap
sudo chown -R ceph:ceph /var/lib/ceph /etc/ceph

sudo systemctl enable --now ceph-mon@ceph-node1
sudo ceph -s
```

You should see `HEALTH_OK` with one monitor in quorum.

---

## Step 5: Start the Manager

The manager provides metrics, the dashboard, and orchestration:

```bash
sudo mkdir -p /var/lib/ceph/mgr/ceph-ceph-node1
sudo ceph auth get-or-create mgr.ceph-node1 \
  mon "allow profile mgr" osd "allow *" mds "allow *" \
  -o /var/lib/ceph/mgr/ceph-ceph-node1/keyring
sudo chown -R ceph:ceph /var/lib/ceph/mgr

sudo systemctl enable --now ceph-mgr@ceph-node1
sudo ceph -s
# Look for: mgr: ceph-node1(active)
```

---

## Step 6: Create OSDs with Loopback Devices

No spare disks? Loopback files work for testing.

**Create the backing files and set up LVM:**

```bash
for i in 0 1 2; do
  sudo dd if=/dev/zero of=/home/ceph-osd-$i.img bs=1M count=10240
  sudo losetup /dev/loop$i /home/ceph-osd-$i.img
  sudo parted -s /dev/loop$i mklabel gpt
  sudo parted -s /dev/loop$i mkpart primary 0% 100%
  sudo pvcreate /dev/loop${i}p1
  sudo vgcreate ceph-vg-$i /dev/loop${i}p1
  sudo lvcreate -n ceph-lv-$i -l 100%FREE ceph-vg-$i
done
```

**Create the OSDs:**

```bash
sudo ceph auth import -i /var/lib/ceph/bootstrap-osd/ceph.keyring

for i in 0 1 2; do
  sudo ceph-volume lvm create --data ceph-vg-$i/ceph-lv-$i
done
```

**Fix the directory naming and start them:**

```bash
for i in 3 4 5; do
  sudo ln -s /var/lib/ceph/osd/ceph-$i /var/lib/ceph/osd/$i
  sudo systemctl start ceph-osd@$i
done

sudo ceph -s
# osd: 3 osds: 3 up, 3 in
```

---

## Step 7: Create CephFS

First, create data and metadata pools:

```bash
sudo ceph osd pool create cephfs_data 32
sudo ceph osd pool create cephfs_metadata 32
sudo ceph fs new cephfs cephfs_metadata cephfs_data
```

Then start the Metadata Server:

```bash
sudo mkdir -p /var/lib/ceph/mds/ceph-ceph-node1
sudo ceph auth get-or-create mds.ceph-node1 \
  mon "allow profile mds" osd "allow *" mds "allow *" \
  -o /var/lib/ceph/mds/ceph-ceph-node1/keyring
sudo chown -R ceph:ceph /var/lib/ceph/mds

sudo systemctl enable --now ceph-mds@ceph-node1
sudo ceph fs ls
# name: cephfs, metadata pool: cephfs_metadata, data pools: [cephfs_data]
```

---

## Step 8: Mount and Test

```bash
# Grab the admin secret
CEPH_SECRET=*** ceph auth get-key client.admin)

# Mount it
sudo mkdir -p /mnt/cephfs
sudo mount -t ceph 192.168.211.133:6789:/ /mnt/cephfs \
  -o name=admin,secret=***

# Write a test file
echo "Hello from CephFS!" | sudo tee /mnt/cephfs/test.txt
cat /mnt/cephfs/test.txt
# Hello from CephFS!

df -h /mnt/cephfs
# 29G available
```

---

## Final Status

```bash
sudo ceph -s
```

```
cluster:
    id:     a1b2c3d4-e5f6-7890-abcd-ef1234567890
    health: HEALTH_WARN
            3 pool(s) have no replicas configured

services:
    mon: 1 daemons, quorum ceph-node1
    mgr: ceph-node1(active)
    mds: 1/1 daemons up
    osd: 3 osds: 3 up, 3 in

data:
    volumes: 1/1 healthy
    pools:   3 pools, 65 pgs
    objects: 2 objects, 449 KiB
    usage:   80 MiB used, 30 GiB / 30 GiB avail
    pgs:     65 active+clean
```

The `HEALTH_WARN` about replicas is normal — we set `osd_pool_default_size = 1` since this is a single-node cluster.

---

## Cleanup

To tear everything down:

```bash
sudo umount /mnt/cephfs
sudo systemctl stop ceph-{mon,mgr,mds}@ceph-node1
for i in 3 4 5; do sudo systemctl stop ceph-osd@$i; done
sudo dnf remove -y ceph-*
sudo rm -rf /var/lib/ceph /etc/ceph
for i in 0 1 2; do
  sudo losetup -d /dev/loop$i
  sudo rm -f /home/ceph-osd-$i.img
done
```

---

## What's Next?

- **Enable the dashboard** — `sudo ceph mgr module enable dashboard`
- **Add more nodes** — Install Ceph on additional VMs and join them to the cluster
- **Set up RBD** — Create block devices for KVM or containers
- **S3-compatible storage** — Deploy RadosGW for S3 API access

---

*Built on CentOS Stream 9 with Ceph Squid 19.2.4. Single-node test cluster — not for production use.*
