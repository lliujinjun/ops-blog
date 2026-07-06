+++
date = '2026-07-06T12:50:00+08:00'
draft = false
title = 'Manual Ceph Installation: A Step-by-Step Walkthrough'
+++

**Environment:** CentOS Stream 9 · Ceph Reef 18.2.8 · Manual (Package-Based) · VMware VM · 3 × 20 GB OSDs

---

Ceph's recommended installation method today is **cephadm** — containers, one command, done. But running through the manual (package-based) installation teaches you what's actually happening under the hood: keyrings, auth, systemd units, CRUSH maps. You'll understand the cluster in a way containerization abstracts away.

This post walks through the full manual install on a single CentOS Stream 9 node, from blank OS to a working cluster with RBD, S3 (RGW), and CephFS — all running as native systemd services.

---

## Why Manual?

| Approach | Abstraction | What You Learn |
|----------|-------------|----------------|
| cephadm | Full | Operations |
| Manual | None | Architecture, auth, internals |

Manual install is also useful for **air-gapped environments**, **custom deployments**, and **troubleshooting** — when the cephadm container won't start, you need to know what it was trying to do.

---

## Prerequisites

**Hardware:** A VM (or physical machine) with:

- 4 vCPUs, 4 GB RAM
- One OS disk (200 GB)
- **Three raw disks** for OSDs — 20 GB each (`/dev/sdb`, `/dev/sdc`, `/dev/sdd`)

**Software:** CentOS Stream 9 (or any EL9 derivative)

**Network:** Static IP (192.168.211.133 in this guide), hostname resolves in `/etc/hosts`

---

## Step 1 — Host Preparation

```bash
sudo hostnamectl set-hostname ceph-node1
echo "192.168.211.133 ceph-node1" | sudo tee -a /etc/hosts
sudo dnf install -y epel-release
```

---

## Step 2 — Install Ceph Packages

```bash
sudo dnf install -y centos-release-ceph-reef
sudo dnf install -y ceph ceph-radosgw
```

This installs everything: `ceph-mon`, `ceph-osd`, `ceph-mgr`, `ceph-mds`, `ceph-radosgw`, `ceph-common`, and the systemd unit files.

Check that the binaries exist:

```bash
which ceph-mon ceph-osd ceph-mgr ceph-mds radosgw
```

---

## Step 3 — Generate Cluster ID and Config

Every Ceph cluster needs a unique **FSID** (UUID). Generate one and create the configuration file:

```bash
FSID=$(uuidgen)

cat > /etc/ceph/ceph.conf << EOF
[global]
fsid = $FSID
mon host = 192.168.211.133
public network = 192.168.211.0/24
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
osd pool default size = 2
osd pool default min size = 1
EOF
```

> **Note:** `osd pool default size = 2` is pre-set for single-node. The default of 3 would need at least 3 different hosts.

---

## Step 4 — Create Keyrings

Ceph uses **cephx** authentication. Every daemon and client needs a keyring.

```bash
# Monitor keyring
sudo ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring \
  --gen-key -n mon. --cap mon 'allow *'

# Admin keyring (used by 'ceph' CLI)
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring \
  --gen-key -n client.admin --cap mon 'allow *' \
  --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'

# Bootstrap keyrings (needed by ceph-volume for OSD provisioning)
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring \
  --gen-key -n client.bootstrap-osd \
  --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'

sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-mds/ceph.keyring \
  --gen-key -n client.bootstrap-mds \
  --cap mon 'profile bootstrap-mds'

sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-rgw/ceph.keyring \
  --gen-key -n client.bootstrap-rgw \
  --cap mon 'profile bootstrap-rgw'

# Add admin keyring to mon keyring
sudo ceph-authtool /etc/ceph/ceph.mon.keyring \
  --import-keyring /etc/ceph/ceph.client.admin.keyring
```

---

## Step 5 — Start the Monitor (MON)

The monitor is the cluster's brain — it maintains the cluster map and manages quorum.

```bash
sudo mkdir -p /var/lib/ceph/mon/ceph-ceph-node1
sudo chown ceph:ceph /var/lib/ceph/mon/ceph-ceph-node1

sudo -u ceph ceph-mon --mkfs -i ceph-node1 \
  --keyring /etc/ceph/ceph.mon.keyring

sudo systemctl enable --now ceph-mon@ceph-node1
```

Verify:

```bash
sudo ceph -s
# Should show 1 mon, HEALTH_WARN (no OSDs yet)
```

---

## Step 6 — Start the Manager (MGR)

The manager provides extra monitoring, the dashboard, and orchestration.

```bash
sudo mkdir -p /var/lib/ceph/mgr/ceph-ceph-node1

sudo ceph auth get-or-create mgr.ceph-node1 \
  mon 'allow profile mgr' osd 'allow *' mds 'allow *' \
  -o /var/lib/ceph/mgr/ceph-ceph-node1/keyring

sudo chown -R ceph:ceph /var/lib/ceph/mgr/ceph-ceph-node1
sudo systemctl enable --now ceph-mgr@ceph-node1
```

---

## Step 7 — Import Bootstrap Keys

The bootstrap keyrings need to be registered with the cluster before we can create OSDs:

```bash
sudo ceph auth import -i /var/lib/ceph/bootstrap-osd/ceph.keyring
sudo ceph auth import -i /var/lib/ceph/bootstrap-mds/ceph.keyring
sudo ceph auth import -i /var/lib/ceph/bootstrap-rgw/ceph.keyring
```

---

## Step 8 — Deploy OSDs

**OSDs** (Object Storage Daemons) are the workers that actually store your data. Each one claims a raw disk.

Use `ceph-volume` to prepare and activate each disk:

```bash
sudo ceph-volume lvm create --data /dev/sdb
sudo ceph-volume lvm create --data /dev/sdc
sudo ceph-volume lvm create --data /dev/sdd
```

This does three things per disk:

1. **Prepares** — creates an LVM volume, formats it as XFS, generates a keyring
2. **Activates** — starts the OSD daemon, mounts the volume
3. **Registers** — adds the OSD to the CRUSH map

Verify:

```bash
sudo ceph -s
# Target: 3 osds: 3 up, 3 in
```

---

## Step 9 — The Single-Node CRUSH Fix

By default, Ceph's CRUSH rule (`replicated_rule`) uses **host** as the failure domain — it wants replicas on different physical hosts. On a single node, PGs stay stuck at `undersized+degraded` and can never reach `active+clean`.

The fix: create a CRUSH rule that replicates across **OSD** devices instead of hosts.

```bash
sudo ceph osd crush rule create-replicated single-node default osd
```

Then apply it to existing pools:

```bash
sudo ceph osd pool set .mgr crush_rule single-node
sudo ceph osd pool set .mgr size 2
```

Set cluster-wide defaults so new pools automatically get the right values:

```bash
sudo ceph config set global osd_pool_default_size 2
sudo ceph config set global osd_pool_default_min_size 1
```

Restart the monitor to clear the `insecure global_id reclaim` warning:

```bash
sudo systemctl restart ceph-mon@ceph-node1
```

At this point, the cluster should reach **HEALTH_OK**:

```
  cluster:
    id:     8a261b37-cb22-4aed-b1ea-37372ef72b2c
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum ceph-node1
    mgr: ceph-node1(active)
    osd: 3 osds: 3 up, 3 in

  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 449 KiB
    usage:   80 MiB used, 60 GiB / 60 GiB avail
    pgs:     1 active+clean
```

---

## Step 10 — Configure RBD (Block Storage)

**RBD** provides block devices that can be mapped as virtual disks, typically used for VM images.

```bash
sudo ceph osd pool create rbd 32 32 replicated single-node
sudo ceph osd pool application enable rbd rbd

# Create a test image
sudo rbd create test-image --size 1G --pool rbd
sudo rbd ls --pool rbd
# -> test-image

# Resize on the fly
sudo rbd resize --pool rbd --image test-image --size 2G
```

---

## Step 11 — Configure RGW (S3 Object Gateway)

**RGW** exposes Ceph storage as an S3-compatible API.

```bash
# Create data directory
sudo mkdir -p /var/lib/ceph/radosgw/ceph-rgw.ceph-node1

# Generate keyring
sudo ceph auth get-or-create client.rgw.ceph-node1 \
  osd 'allow rwx' mon 'allow rw' \
  -o /var/lib/ceph/radosgw/ceph-rgw.ceph-node1/keyring

sudo chown -R ceph:ceph /var/lib/ceph/radosgw/ceph-rgw.ceph-node1

# Start the daemon
sudo systemctl enable --now ceph-radosgw@rgw.ceph-node1
```

Fix the CRUSH rule for the automatically-created RGW pools:

```bash
for pool in $(sudo ceph osd pool ls | grep -E '^\.rgw|^default\.rgw'); do
  sudo ceph osd pool set $pool crush_rule single-node
  sudo ceph osd pool set $pool size 2
done
```

Create an S3 user:

```bash
sudo radosgw-admin user create --uid=testuser --display-name='Test User'
```

Output includes access and secret keys:

```json
"access_key": "J8GIF77OMGPJ0O3DIKPS",
"secret_key": "S9HQTzbgDTirof1s3Ciw15PXHDIpD7B543DBLXnC"
```

---

## Step 12 — Configure CephFS (Shared Filesystem)

**CephFS** provides a POSIX-compatible distributed filesystem.

```bash
# Create metadata and data pools
sudo ceph osd pool create cephfs.myfs.meta 32 32 replicated single-node
sudo ceph osd pool create cephfs.myfs.data 32 32 replicated single-node
sudo ceph osd pool set cephfs.myfs.meta size 2
sudo ceph osd pool set cephfs.myfs.data size 2

# Create the filesystem
sudo ceph fs new myfs cephfs.myfs.meta cephfs.myfs.data

# Deploy MDS
sudo mkdir -p /var/lib/ceph/mds/ceph-ceph-node1
sudo ceph auth get-or-create mds.ceph-node1 \
  osd 'allow rwx' mds 'allow' mon 'allow profile mds' \
  -o /var/lib/ceph/mds/ceph-ceph-node1/keyring
sudo chown -R ceph:ceph /var/lib/ceph/mds/ceph-ceph-node1
sudo systemctl enable --now ceph-mds@ceph-node1
```

Install the FUSE client and test the mount:

```bash
sudo dnf install -y ceph-fuse
sudo mkdir -p /mnt/myfs
sudo ceph-fuse /mnt/myfs
echo "hello from manual cephfs" | sudo tee /mnt/myfs/hello.txt
sudo cat /mnt/myfs/hello.txt
# -> hello from manual cephfs
sudo umount /mnt/myfs
```

---

## Final Cluster Status

```
  cluster:
    id:     8a261b37-cb22-4aed-b1ea-37372ef72b2c
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum ceph-node1
    mgr: ceph-node1(active, since 2m)
    mds: 1/1 daemons up
    osd: 3 osds: 3 up, 3 in
    rgw: 1 daemon active (1 hosts, 1 zones)

  data:
    volumes: 1/1 healthy
    pools:   8 pools, 225 pgs
    objects: 218 objects, 456 KiB
    usage:   107 MiB used, 60 GiB / 60 GiB avail
    pgs:     225 active+clean
```

A fully functional cluster with **RBD block storage**, **S3 object gateway**, and **CephFS shared filesystem** — all running as native systemd services.

---

## What the Manual Approach Taught Me

Going through this manually reveals several things the cephadm bootstrap hides:

1. **Keyrings are the backbone** — every daemon needs a properly scoped auth key. Get this wrong and nothing works.
2. **ceph-volume does the OSD heavy lifting** — it handles LVM, XFS, and the systemd unit in one command.
3. **The CRUSH rule is architecture-critical** — without the `single-node` rule, a one-node cluster can never reach `active+clean`.
4. **Systemd integration is solid** — every daemon has a well-designed `@.service` template. Enabling and starting is identical to any other system service.
5. **You appreciate cephadm more** — it isn't magic, it's just automating exactly these 12 steps.

---

## Cleanup

The test resources created during this walkthrough:

```bash
# Remove RBD image
sudo rbd rm --pool rbd test-image

# Remove CephFS test mount
sudo rm -f /mnt/myfs/hello.txt

# Remove S3 test user
sudo radosgw-admin user rm --uid=testuser --purge-data
```

---

*Manual or cephadm — now you know what's going on either way.*
