+++
date = '2026-07-06T14:00:00+08:00'
draft = false
title = 'Quick Manual Ceph Install: MON, MGR, OSD in 10 Minutes'
+++

**Environment:** CentOS Stream 9 · Ceph Reef 18.2.8 · Manual (Package-Based) · VMware VM · 3 × 20 GB OSDs

---

This is the no-fluff version: from a blank CentOS Stream 9 VM to a working Ceph cluster with MON, MGR, and 3 OSDs in about 10 minutes. No RGW, no CephFS, no dashboard — just the bare minimum for a running cluster.

The goal isn't production readiness. It's seeing all three mandatory daemons come up, reach quorum, and store data.

---

## Prerequisites

- **CentOS Stream 9** VM (or EL9 derivative)
- **Three raw disks** for OSDs (`/dev/sdb`, `/dev/sdc`, `/dev/sdd`)
- Static IP or stable DHCP

---

## The Commands

Everything below is one sequence — run them in order, no detours.

### 1. Host prep

```bash
sudo hostnamectl set-hostname ceph-node1
echo "192.168.211.133 ceph-node1" | sudo tee -a /etc/hosts
```

### 2. Install packages

```bash
sudo dnf install -y epel-release centos-release-ceph-reef
sudo dnf install -y ceph
```

### 3. Create config

```bash
FSID=$(uuidgen)
sudo mkdir -p /etc/ceph

sudo tee /etc/ceph/ceph.conf << EOF
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

### 4. Create keyrings

```bash
# Monitor keyring
sudo ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring \
  --gen-key -n mon. --cap mon 'allow *'

# Admin keyring
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring \
  --gen-key -n client.admin --cap mon 'allow *' \
  --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'

# Bootstrap keyrings
sudo mkdir -p /var/lib/ceph/bootstrap-{osd,mds,rgw}

sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring \
  --gen-key -n client.bootstrap-osd \
  --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'

sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-mds/ceph.keyring \
  --gen-key -n client.bootstrap-mds \
  --cap mon 'profile bootstrap-mds'

sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-rgw/ceph.keyring \
  --gen-key -n client.bootstrap-rgw \
  --cap mon 'profile bootstrap-rgw'

# Embed admin keyring in mon keyring
sudo ceph-authtool /etc/ceph/ceph.mon.keyring \
  --import-keyring /etc/ceph/ceph.client.admin.keyring
```

### 5. Start the Monitor

```bash
sudo mkdir -p /var/lib/ceph/mon/ceph-ceph-node1
sudo chown ceph:ceph /var/lib/ceph/mon/ceph-ceph-node1
sudo chown ceph:ceph /etc/ceph/ceph.mon.keyring
sudo chown ceph:ceph /etc/ceph/ceph.client.admin.keyring

sudo -u ceph ceph-mon --mkfs -i ceph-node1 --keyring /etc/ceph/ceph.mon.keyring
sudo systemctl enable --now ceph-mon@ceph-node1

# Verify
sudo ceph -s
# Target: mon: 1 daemons, quorum ceph-node1
```

> **Permission gotcha:** The `ceph-mon --mkfs` and the systemd unit both run as the `ceph` user. If keyring files are owned by root, mkfs fails with "Permission denied." Fix with `chown ceph:ceph`.

### 6. Start the Manager

```bash
sudo mkdir -p /var/lib/ceph/mgr/ceph-ceph-node1
sudo ceph auth get-or-create mgr.ceph-node1 \
  mon 'allow profile mgr' osd 'allow *' mds 'allow *' \
  -o /var/lib/ceph/mgr/ceph-ceph-node1/keyring
sudo chown -R ceph:ceph /var/lib/ceph/mgr/ceph-ceph-node1
sudo systemctl enable --now ceph-mgr@ceph-node1
```

### 7. Import bootstrap keys

```bash
sudo ceph auth import -i /var/lib/ceph/bootstrap-osd/ceph.keyring
sudo ceph auth import -i /var/lib/ceph/bootstrap-mds/ceph.keyring
sudo ceph auth import -i /var/lib/ceph/bootstrap-rgw/ceph.keyring
```

### 8. Deploy OSDs

```bash
sudo ceph-volume lvm create --data /dev/sdb
sudo ceph-volume lvm create --data /dev/sdc
sudo ceph-volume lvm create --data /dev/sdd
```

### 9. Fix single-node CRUSH rule

```bash
sudo ceph osd crush rule create-replicated single-node default osd
sudo ceph osd pool set .mgr crush_rule single-node
sudo ceph osd pool set .mgr size 2
sudo ceph config set global osd_pool_default_size 2
sudo ceph config set global osd_pool_default_min_size 1
```

---

## What You Should See

```bash
$ sudo ceph -s
  cluster:
    id:     5b6265b7-8230-4fc5-a137-3729f751265e
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

Key things to verify:

| Field | Expected | Meaning |
|-------|----------|---------|
| `health` | `HEALTH_OK` | No warnings or errors |
| `quorum ceph-node1` | Your hostname | MON has consensus (trivial with 1 MON) |
| `mgr: ... active` | Your hostname | Manager is serving metrics |
| `3 up, 3 in` | All 3 OSDs | Every disk is claimed and running |
| `60 GiB avail` | Sum of disk sizes | All 3 × 20 GB accounted for |
| `1 active+clean` | `active+clean` | The data placement worked |

---

## What's Missing (and Why It's Fine)

This cluster only has MON, MGR, and OSDs. That's enough for the cluster to be **healthy** — but not useful yet:

- **No RBD pool** — can't create block images
- **No RGW** — no S3 API
- **No MDS/CephFS** — no shared filesystem
- **No Dashboard** — the MGR has the module, but you'd need to enable and configure it

Those are all `ceph osd pool create` + `systemctl enable` commands away. But the core Ceph architecture — the brain (MON), the dashboard (MGR), and the workers (OSDs) — is complete.

---

## Common Pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Permission denied` on mkfs | Keyring owned by root | `chown ceph:ceph` on keyring files |
| MON won't start after mkfs | `systemd` rate limit from failed attempts | `systemctl reset-failed ceph-mon@...` |
| OSDs stuck `undersized+degraded` | Default CRUSH rule needs 3 hosts | Create `single-node` rule with `type osd` |
| `insecure global_id reclaim` | New Reef security feature | `ceph config set mon auth_allow_insecure_global_id_reclaim false` + restart mon |

---

## What Each Daemon Does

| Daemon | Process | Data Directory | Port |
|--------|---------|----------------|------|
| **MON** | `ceph-mon -f --cluster ceph --id ceph-node1` | `/var/lib/ceph/mon/ceph-ceph-node1/` | 3300, 6789 |
| **MGR** | `ceph-mgr -f --cluster ceph --id ceph-node1` | `/var/lib/ceph/mgr/ceph-ceph-node1/` | 8443, 9283 |
| **OSD.0** | `ceph-osd -f --cluster ceph --id 0` | `/var/lib/ceph/osd/ceph-0/` | 6800-7300 |

All run as the `ceph` user, all managed by systemd templates (`ceph-<daemon>@.service`).

---

## Cleanup

To tear down and start fresh:

```bash
sudo systemctl stop ceph-mon@ceph-node1 ceph-mgr@ceph-node1
sudo systemctl stop ceph-osd@{0,1,2}
sudo rm -rf /var/lib/ceph/* /etc/ceph/*
sudo dnf remove -y ceph
```

---

*MON, MGR, OSD — the skeleton of every Ceph cluster. Everything else is optional.*
