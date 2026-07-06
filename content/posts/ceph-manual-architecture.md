+++
date = '2026-07-06T13:40:00+08:00'
draft = false
title = 'Ceph Manual Installation: The Architecture Behind the Daemons'
+++

**Environment:** CentOS Stream 9 · Ceph Reef 18.2.8 · Manual (Package-Based)

---

The cephadm approach installs Ceph in one command. But every cephadm bootstrap is doing the same dozen steps under the hood — creating keyrings, initializing monitors, generating configs. This post explains the **architecture** of those steps: why each daemon exists, how they talk to each other, and what actually happens when you run each command.

---

## The Three Mandatory Daemons

Every Ceph cluster needs three daemon types to function:

| Daemon | Role | Analogy |
|--------|------|---------|
| **MON** | Cluster brain, quorum, cluster map | The conductor |
| **MGR** | Metrics, dashboard, balancing | The dashboard |
| **OSD** | Stores data, replication, recovery | The workers |

Plus optional ones for specific use cases:

| Daemon | Role |
|--------|------|
| **MDS** | CephFS filesystem metadata |
| **RGW** | S3-compatible object gateway |

---

## Step 1: The Config File — The Rosetta Stone

```ini
[global]
fsid = 8a261b37-cb22-4aed-b1ea-37372ef72b2c
mon host = 192.168.211.133
public network = 192.168.211.0/24
```

Every Ceph daemon reads `/etc/ceph/ceph.conf` on startup. It tells them:

- **Who they are** — the `fsid` identifies the cluster. If two clusters accidentally merge, the FSID mismatch prevents corruption.
- **Where the MON is** — `mon host` gives the bootstrap address. Daemons connect here first, then learn the full cluster map.
- **What network to use** — `public network` separates client traffic from replication traffic in multi-NIC setups.
- **How to authenticate** — `auth cluster/service/client required = cephx` enforces cryptographic authentication on every connection.

Without this file, no daemon can find the cluster.

---

## Step 2: Keyrings — Who Gets In and What They Can Do

Ceph uses **cephx** authentication, similar to Kerberos. Every daemon and client authenticates with a keyring — a secret key plus a set of capabilities (caps).

### The Monitor Keyring

```bash
ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring \
  --gen-key -n mon. --cap mon 'allow *'
```

This is the **root key**. It's only used during `ceph-mon --mkfs` to initialize the monitor's database. Once the MON is running, it has its own embedded copy.

### The Admin Keyring

```bash
ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring \
  --gen-key -n client.admin --cap mon 'allow *' \
  --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
```

This is what the `ceph` CLI uses. The caps give it full access to everything. In production, you'd create separate keyrings for different users with limited caps:

```bash
client.backup → mon 'allow r' osd 'allow r'
client.vm-storage → mon 'allow r' osd 'allow rwx pool=vms'
```

### The Bootstrap Keyrings

```bash
client.bootstrap-osd  → mon 'profile bootstrap-osd'
client.bootstrap-mds  → mon 'profile bootstrap-mds'
client.bootstrap-rgw  → mon 'profile bootstrap-rgw'
```

These are a **privilege escalation mechanism**. When `ceph-volume` creates a new OSD, it uses the bootstrap key to authenticate temporarily. The MON then issues a permanent OSD key. This way, the admin key isn't needed on every OSD node.

### The Import Trick

```bash
ceph-authtool ceph.mon.keyring --import-keyring ceph.client.admin.keyring
```

This embeds the admin keyring inside the mon keyring before `mkfs`. After that, the MON knows about the admin user from birth — you can run `ceph -s` immediately after starting it.

---

## Step 3: The Monitor — The Cluster Brain

```bash
ceph-mon --mkfs -i ceph-node1 --keyring /etc/ceph/ceph.mon.keyring
```

`--mkfs` creates the MON's local data store at `/var/lib/ceph/mon/ceph-ceph-node1/`. This is a **key-value database** (RocksDB, embedded) that holds:

| Data | Contents |
|------|----------|
| **MonMap** | Which MONs exist, their addresses |
| **OSDMap** | Which OSDs exist, their state (up/down/in/out), weights |
| **CRUSHMap** | The topology rules for data placement |
| **Auth database** | All keyrings and their caps |
| **Config database** | Cluster-wide settings |

**Why is the MON started first?** Because nothing else can function without the cluster map. Before the MON, there is no cluster. After the MON, you have:

```
mon: 1 daemons, quorum ceph-node1
```

**Quorum** means the MONs agree on cluster state. With one MON, quorum is always achieved. With three MONs, you need two to agree (majority). This is why MONs use **Paxos** — the distributed consensus protocol that prevents split-brain.

---

## Step 4: The Manager — The Dashboard

```bash
ceph auth get-or-create mgr.ceph-node1 \
  mon 'allow profile mgr' osd 'allow *' mds 'allow *' \
  -o /var/lib/ceph/mgr/ceph-ceph-node1/keyring
```

The Manager is technically optional but practically mandatory. It handles:

- **Dashboard** — the web UI at port 8443
- **Prometheus metrics** — endpoint at port 9283
- **PG autoscaling** — automatically adjusts PG counts based on pool usage
- **Balancer** — rebalances PGs across OSDs
- **Orchestration** — the cephadm module that manages daemon deployment

When the MGR starts, it registers with the MON and becomes the active manager:

```
mgr: ceph-node1(active, since 6s)
```

The MON holds a leader election and only one MGR is active at a time.

---

## Step 5: OSDs — The Storage Workers

```bash
ceph-volume lvm create --data /dev/sdb
```

This is the most complex step automated by a single command. Under the hood:

1. **LVM setup** — creates a logical volume on the disk. Ceph uses LVM2 for OSDs because it supports device management, snapshots, and encryption.
2. **XFS format** — formats the LV as XFS, Ceph's recommended local filesystem for its balance of performance and features.
3. **Mount** — mounts the XFS volume at `/var/lib/ceph/osd/ceph-<OSD-ID>/`.
4. **Keyring generation** — contacts the MON via the bootstrap-osd key, gets a permanent OSD identity and key.
5. **Systemd unit** — creates and starts `ceph-osd@<OSD-ID>.service`.

### How the bootstrap flow works

```
ceph-volume → MON (using bootstrap-osd key)
    → "Hello, I'm a new OSD candidate"
MON → ceph-volume
    → "OK, here's OSD ID 0, here's your permanent key"
ceph-volume → /dev/sdb
    → Creates LV, XFS, mounts, writes key, starts daemon
OSD.0 → MON
    → "I'm alive, here's my key"
MON → OSD.0
    → "Welcome, here's the cluster map"
```

After creation:

```
osd: 3 osds: 3 up (since 23s), 3 in (since 32s)
  up = daemon is running
  in = OSD is part of the cluster and accepting data
```

---

## Step 6: CRUSH — Where Does Data Go?

CRUSH (**Controlled Replication Under Scalable Hashing**) is Ceph's data placement algorithm. It's what makes Ceph decentralized — no central metadata server tells you where data lives.

### How CRUSH works

```
Object "photo.jpg" → hash("photo.jpg") → hash value
                  → CRUSH(hash value, cluster map, rule)
                  → OSD set: [Osd.2, Osd.0]
```

CRUSH deterministically maps every object to a set of OSDs based on:

1. The object name (hashed)
2. The cluster topology (which hosts, which racks, which OSDs)
3. The replication rule (how many copies, where to place them)

### The single-node problem

The default rule is:

```
rule replicated_rule {
    ruleset 0
    type replicated
    min_size 1
    max_size 10
    step take default
    step chooseleaf firstn 0 type host
    step emit
}
```

`step chooseleaf firstn 0 type host` means: pick N different hosts, then pick one OSD per host. On a single node, Ceph can never find 3 different hosts, so PGs get stuck at `undersized+degraded`.

### The fix

```bash
ceph osd crush rule create-replicated single-node default osd
```

This creates:

```
rule single-node {
    ruleset 1
    type replicated
    min_size 1
    max_size 10
    step take default
    step chooseleaf firstn 0 type osd
    step emit
}
```

Now CRUSH picks N different **OSDs** instead of N different hosts. With 3 OSDs and `size=2`, data can be placed.

### Visualizing placement

```
Each PG has 2 replicas on a 3-OSD cluster:

PG 1.0 → Osd.0 + Osd.2
PG 1.1 → Osd.1 + Osd.0
PG 1.2 → Osd.2 + Osd.1
...

Every OSD has a copy of roughly 1/3 of the data.
Losing any one OSD means the cluster still has 2 copies of everything.
```

---

## Step 7: The RGW — S3 Object Gateway

RGW is a translation layer. It speaks the S3 protocol (HTTP/REST) and converts every request into RADOS operations:

```
PUT /my-bucket/photo.jpg
  → RGW receives HTTP request
  → RGW looks up bucket info in .rgw.root pool
  → RGW writes object data to default.rgw.buckets.data pool
  → RGW updates bucket index in default.rgw.buckets.index pool
  → RGW responds "201 Created"
```

From Ceph's perspective, an S3 object is just a RADOS object with a specific layout. The RGW manages:

- **User accounts** and their keys
- **Bucket metadata** (who owns it, what region)
- **Object index** (list objects in a bucket)
- **Multipart uploads** (for large objects)
- **Versioning** and **lifecycle policies**

### RGW pools

When RGW starts, it creates several pools automatically:

| Pool | Purpose |
|------|---------|
| `.rgw.root` | Realm/zonegroup/zone config |
| `default.rgw.control` | Heartbeat and control |
| `default.rgw.meta` | User and bucket metadata |
| `default.rgw.log` | Usage and audit logs |
| `default.rgw.buckets.index` | Bucket object listings |
| `default.rgw.buckets.data` | The actual objects |

Without fixing their CRUSH rules, these pools default to `replicated_rule` (failure domain = host). On a single node, their PGs can never become `active+clean`.

---

## Step 8: The MDS — CephFS Metadata

CephFS has a unique architecture: **data goes directly from clients to OSDs**, but **metadata goes through the MDS**.

```
ceph-fuse /mnt/myfs
  → Client opens /mnt/myfs/hello.txt
  → Client asks MDS: "What OSDs hold the data for hello.txt?"
  → MDS looks up metadata pool → returns OSD locations
  → Client reads data directly from OSDs (no MDS bottleneck)
```

This separation means:

- **Metadata** uses a fast pool (often on SSDs/NVMe)
- **Data** uses a capacious pool (often on HDDs)
- The MDS caches metadata in RAM for fast lookups
- Multiple MDS daemons can scale metadata throughput

To function, CephFS needs:

```
Pool 1: cephfs.myfs.meta — stores filenames, permissions, directory structure
Pool 2: cephfs.myfs.data — stores file contents
Daemon: mds.myfs — serves metadata lookups
```

The MDS daemon is lightweight. What matters is the RAM cache — larger cache means fewer metadata operations hit the slow pool.

---

## The Big Picture

```
                    ┌──────────────────┐
                    │   ceph CLI       │
                    │  (admin keyring) │
                    └────────┬─────────┘
                             │
                             ▼
┌─────────────────────────────────────────────┐
│        MON (mon.ceph-node1)                 │
│  - Cluster map (MonMap, OSDMap)            │
│  - Auth database (who can do what)         │
│  - Config database (settings)              │
│  - Paxos quorum (consensus)                │
└────┬──────────┬──────────┬──────────────────┘
     │          │          │
     ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌───────────┐
│ MGR    │ │ MDS    │ │ RGW       │
│ Metrics│ │CephFS  │ │ S3 REST   │
│ Dashbd │ │Meta    │ │ gateway   │
│ Balance│ │Cache   │ │           │
└───┬────┘ └────────┘ └───────────┘
    │
    ▼
┌───────────────────────────────────┐
│   OSD.0    OSD.1    OSD.2        │
│  /dev/sdb  /dev/sdc  /dev/sdd    │
│  20 GB     20 GB     20 GB       │
│                                   │
│  60 GiB total, 2x replication    │
│  ≈ 29 GiB usable                 │
└───────────────────────────────────┘
```

Every daemon follows the same startup flow:

1. Read config → find the MON address
2. Present keyring → authenticate
3. Receive cluster map → learn who's who
4. Start serving → register as active

## Summary

| Step | Command | What It Creates |
|------|---------|-----------------|
| Config | `ceph.conf` | Cluster identity, network, auth |
| Keyrings | `ceph-authtool` | Authentication secrets + caps |
| MON | `ceph-mon --mkfs` | Cluster brain, quorum, cluster map |
| MGR | `ceph-mgr` | Dashboard, metrics, balancer |
| OSD | `ceph-volume lvm create` | One storage worker per disk |
| CRUSH fix | `ceph osd crush rule create` | Single-node data placement rule |
| RGW | `ceph-radosgw` | S3-compatible HTTP gateway |
| MDS | `ceph-mds` | CephFS metadata server |

The manual install teaches you what cephadm does in one command. The architecture is the same either way — containers or native systemd — and understanding it makes you better at debugging both.
