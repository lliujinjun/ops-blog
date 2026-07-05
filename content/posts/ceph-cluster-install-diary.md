---
title: "📓 Ceph Cluster Install Diary: A Real Session on CentOS Stream 9"
date: 2026-07-05T13:04:30+08:00
draft: false
description: "A real-time log of installing Ceph Squid 19.2.4 on a single CentOS Stream 9 VM — including the mistakes, the debugging, and the 'why won't you start' moments."
tags: ["ceph", "storage", "centos", "homelab", "diary"]
categories: ["storage"]
---

If you read the clean guide and thought "it probably didn't work the first time" — you're right. Here's the real session.

<!--more-->

---

## The Setup

One CentOS Stream 9 VM, 3.5GB RAM, 200GB disk. The goal: a single-node Ceph cluster with CephFS.

I SSH'd in from my WSL2 machine through a port-forwarded SSH config. The VM hostname was `localhost.localdomain` — that had to change first.

---

## Installing the Packages

I went for the official Ceph Squid repo. `dnf install` was straightforward... until it wasn't:

```
Error: conflicting requests
 - nothing provides lua-devel needed by ceph-2:19.2.4-0.el9.x86_64 from Ceph
```

The full `ceph` meta-package pulled in development dependencies that didn't exist. Quick pivot — install the individual component packages instead:

```
ceph-mon ceph-mgr ceph-osd ceph-mds ceph-common ceph-radosgw ceph-selinux
```

That worked. But the GPG key for the Ceph repo returned a 404, so `--nogpgcheck` it was.

---

## First Problem: The Monitor Won't Start

After creating the config and all the key rings, I tried to start the monitor. It fired up, but `ceph -s` showed HEALTH_WARN:

```
mon is allowing insecure global_id reclaim
1 monitors have not enabled msgr2
```

The fix was straightforward:

```bash
sudo ceph mon enable-msgr2
sudo ceph config set mon auth_allow_insecure_global_id_reclaim false
sudo systemctl restart ceph-mon@ceph-node1
```

After restart, the warnings cleared. The monitor was happy.

---

## Second Problem: The Manager Won't Authenticate

The manager started, but `ceph -s` kept saying `mgr: no daemons active`. The log said:

```
Module ... has missing NOTIFY_TYPES member
```

Those were just warnings, not the real issue. The real problem? The mgr keyring directory didn't exist when I created the key, so the key was never written. Creating the directory first and re-running `ceph auth get-or-create` fixed it.

---

## Third Problem: OSDs Are Not Starting

This was the frustrating one. I created 3 loopback files (10GB each), partitioned them, and ran `ceph-volume lvm create`. It reported success, but every OSD daemon crashed immediately:

```
auth: unable to find a keyring on /var/lib/ceph/osd/3/keyring
failed to fetch mon config (--no-mon-config to skip)
```

The OSD was looking for its keyring at `/var/lib/ceph/osd/3/keyring`, but `ceph-volume` put the OSD data at `/var/lib/ceph/osd/ceph-3/`. The directory name has a `ceph-` prefix.

The pre-start script handles this correctly (it searches `ceph-$id`), but the daemon itself looks at `$id` without the prefix. The fix was a symlink:

```bash
sudo ln -s /var/lib/ceph/osd/ceph-3 /var/lib/ceph/osd/3
```

After that, all 3 OSDs came up immediately.

---

## Fourth Problem: MDS Keyring

The Metadata Server is needed for CephFS. I created the key, started the service, and it crashed with exit code 1. Same story as the manager — the directory didn't exist when the key was generated. Creating `mkdir -p /var/lib/ceph/mds/ceph-ceph-node1` before running `ceph auth get-or-create` fixed it.

---

## The Moment It All Worked

After all the fixes, the cluster looked like this:

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

The HEALTH_WARN is expected — single-node cluster with `osd_pool_default_size = 1`.

---

## Mounting CephFS

```bash
sudo mkdir -p /mnt/cephfs
sudo mount -t ceph 192.168.211.133:6789:/ /mnt/cephfs \
  -o name=admin,secret=***
echo "Hello from CephFS!" | sudo tee /mnt/cephfs/test.txt
```

It worked. First byte written to the Ceph cluster.

---

## Lessons Learned

1. **Install component packages, not the meta-package** — the `ceph` metapackage has broken deps on CentOS 9
2. **Always create the directory before running `ceph auth get-or-create`** — forgot this twice
3. **ceph-volume names OSD dirs `ceph-N` but the daemon expects `N`** — symlink it
4. **3.5GB RAM is tight** — the manager alone peaks at 390MB. Production clusters need more
5. **The GPG key URL returns 404** — use `--nogpgcheck` or find the correct key URL

---

## Time Spent

About 15 minutes of actual install time, plus debugging the OSD symlink issue. The loopback file creation (3 x 10GB) took the longest at ~40 seconds per file.

---

*Installed on CentOS Stream 9 with Ceph Squid 19.2.4. Single-node test cluster.*
