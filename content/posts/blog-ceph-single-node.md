+++
date = '2026-07-06T10:00:00+08:00'
draft = false
title = 'Running Ceph on a Single Node: A Practical Walkthrough'
+++

**Environment:** CentOS Stream 9 · Ceph Reef 18.2.8 · cephadm · VMware VM

---

Ceph is famously a distributed storage system. Most guides start at three nodes and go up from there. But what if you only have one VM to experiment with — or need a compact storage backend for a lab, edge deployment, or proof-of-concept?

This post walks through installing Ceph Reef on a single CentOS Stream 9 node, with three virtual disks, containerized via cephadm. We'll cover the gotchas, the CRUSH trick that makes single-node work, and what "production" really means in this context.

---

## Why Single-Node Ceph?

Let's be honest about why you'd do this:

- **Learning.** You want to understand Ceph internals — pools, PGs, CRUSH maps, OSD lifecycle — without needing a rack of hardware.
- **CI/CD pipelines.** Testing Ceph integrations in a disposable VM before deploying to a real cluster.
- **Edge / small-scale.** A single beefy node with multiple disks can still benefit from Ceph's object storage semantics, even without host-level redundancy.

The trade-off is real: you lose the distributed redundancy that makes Ceph famous. A motherboard failure takes your data with it. Know that going in.

---

## Prerequisites

### Hardware

Our test VM (VMware) specs:

- **4 vCPUs**
- **4 GB RAM**
- **200 GB OS disk** (partitioned for OS)
- **3 × 20 GB raw disks** (sdb, sdc, sdd — dedicated to OSDs)

> **Heads up:** Ceph *needs* raw block devices for OSDs. You can't share the OS disk. If you're on a VM, add three small virtual disks before starting.

### Software

- CentOS Stream 9 (any RHEL 9 derivative works — Rocky, Alma)
- Podman (comes with CentOS 9)
- Chronyd for time sync

### Networking

- Static IP or stable DHCP reservation
- Firewall ports open: 3300, 6789, 6800-7300, 8443, 8080
- Hostname resolves to the node's IP in `/etc/hosts`

---

## Step 1 — Prepare the Host

```bash
# Set a meaningful hostname
sudo hostnamectl set-hostname ceph-node1

# Add it to /etc/hosts so Ceph can find itself
echo "192.168.211.133 ceph-node1" | sudo tee -a /etc/hosts

# Ensure time sync is running
sudo systemctl enable --now chronyd

# Open firewall ports
sudo firewall-cmd --zone=public \
  --add-port=3300/tcp --add-port=6789/tcp \
  --add-port=6800-7300/tcp --add-port=8443/tcp \
  --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

---

## Step 2 — Install cephadm

CentOS Stream 9 provides a `centos-release-ceph-reef` package that adds the official Ceph repository:

```bash
sudo dnf install -y centos-release-ceph-reef
sudo dnf install -y cephadm
```

This gives you the `cephadm` command, which manages everything via containers.

---

## Step 3 — Bootstrap the Cluster

```bash
sudo cephadm bootstrap --mon-ip 192.168.211.133
```

This single command:

1. Pulls the `quay.io/ceph/ceph:v18` container image (~1.4 GB)
2. Deploys the first **Monitor (MON)** daemon
3. Deploys the first **Manager (MGR)** daemon
4. Generates SSH keys, config files, and the admin keyring
5. Sets up the Ceph Dashboard with auto-generated credentials

Expect this to take a few minutes — the image pull is the bottleneck, especially on slower connections.

> **Pro tip:** If you lose the terminal output, the dashboard password is logged in `/var/log/ceph/cephadm.log`.

After successful bootstrap, check the cluster:

```bash
sudo cephadm shell -- ceph -s
```

You should see a single MON, a single MGR, and `HEALTH_WARN` because there are zero OSDs.

---

## Step 4 — Deploy OSDs

List available devices:

```bash
sudo cephadm shell -- ceph orch device ls
```

If your raw disks are visible, deploy all unused devices:

```bash
sudo cephadm shell -- ceph orch apply osd --all-available-devices
```

After a few seconds, check OSD status:

```bash
sudo cephadm shell -- ceph -s
```

Target: **3 OSDs: 3 up, 3 in**.

---

## Step 5 — The CRUSH Workaround (This Is the Tricky Part)

Ceph's default replication rule (`replicated_rule`) uses `host` as the failure domain. It expects replicas on **different physical hosts**. With all OSDs on one node, PGs stay stuck at `undersized+peered` — they can never reach `active+clean`.

The fix: create a CRUSH rule that replicates across **OSDs** instead of hosts.

```bash
# Create a new rule with 'osd' as the failure domain
sudo cephadm shell -- ceph osd crush rule create-replicated single-node default osd

# Apply it to existing pools (the .mgr pool in this case)
sudo cephadm shell -- ceph osd pool set .mgr crush_rule single-node

# Lower pool size to match available OSDs
sudo cephadm shell -- ceph osd pool set .mgr size 2

# Set cluster-wide defaults for future pools
sudo cephadm shell -- ceph config set global osd_pool_default_size 2
sudo cephadm shell -- ceph config set global osd_pool_default_min_size 1
```

After this, the PG should transition to `active+clean` and the cluster reaches `HEALTH_OK`.

### What the rules mean

| Setting | Value | Meaning |
|---------|-------|---------|
| `size` | 2 | Keep 2 copies of each object |
| `min_size` | 1 | Allow reads/writes even if only 1 copy remains |

With 3 OSDs and `size=2`, you can lose any **1 OSD** and the cluster stays healthy.

---

## Step 6 — Access the Dashboard

```
URL:      https://192.168.211.133:8443/
Username: admin
Password: <check bootstrap output or /var/log/ceph/cephadm.log>
```

The dashboard gives you real-time cluster health, OSD status, performance graphs, and pool management — all from your browser.

---

## Step 7 — Configure RBD (Block Storage)

RADOS Block Device (RBD) provides raw block storage — perfect for VM disk images or database volumes.

### Create an RBD pool

```bash
# Create the pool with the single-node crush rule
sudo cephadm shell -- ceph osd pool create rbd 32 32 replicated single-node
sudo cephadm shell -- ceph osd pool application enable rbd rbd
```

### Create and test a block image

```bash
# Create a 1 GiB image
sudo cephadm shell -- rbd create test-image --size 1G --pool rbd

# Verify
sudo cephadm shell -- rbd ls --pool rbd
sudo cephadm shell -- rbd info test-image --pool rbd

# Resize on the fly
sudo cephadm shell -- rbd resize --pool rbd --image test-image --size 2G

# Snapshot support
sudo cephadm shell -- rbd snap create --pool rbd --image test-image --snap test-snap
```

> **Note:** Mapping the RBD image to a kernel block device requires the `rbd` kernel module or `rbd-nbd`. On VMs without native kernel RBD support, use `rbd-nbd` inside the cephadm container:
> ```bash
> sudo cephadm shell -- rbd-nbd map rbd/test-image
> # -> /dev/nbd0 (2 GiB block device)
> ```

When you're done testing:
```bash
sudo cephadm shell -- rbd-nbd unmap rbd/test-image
sudo cephadm shell -- rbd snap purge --pool rbd --image test-image
sudo cephadm shell -- rbd rm --pool rbd test-image
```

---

## Step 8 — Configure RGW (S3 Object Gateway)

Ceph's RADOS Gateway (RGW) exposes an S3-compatible API. Just a single command to deploy:

```bash
sudo cephadm shell -- ceph orch apply rgw my-store
```

This deploys two RGW daemons on ports **80** and **81**, with internal pools created automatically.

### Create an S3 user

```bash
sudo cephadm shell -- radosgw-admin user create \
  --uid=testuser --display-name="Test User"
```

Output includes your credentials:

```json
{
    "keys": [{
        "access_key": "X4CPL3L1HHUS7VUVVAKK",
        "secret_key": "TaUVcmnYudRlMCB8Bsn5I4Lrpl6otYioBePqKlYM"
    }]
}
```

### Verify S3 access

Test with `curl` to confirm the endpoint responds:

```bash
curl http://192.168.211.133:80/
# <?xml version="1.0"...><ListAllMyBucketsResult>...
```

Or use `radosgw-admin` to manage buckets:

```bash
# List buckets
sudo cephadm shell -- radosgw-admin bucket list

# Create a bucket
# (Buckets are normally created via S3 API, but admin can verify existence)
sudo cephadm shell -- radosgw-admin bucket stats --bucket=my-bucket
```

For programmatic access, use `boto3` with the correct configuration:

```python
import boto3, botocore

session = boto3.session.Session(
    aws_access_key_id='X4CPL3L1HHUS7VUVVAKK',
    aws_secret_access_key='...',
    region_name='us-east-1'
)

client = session.client('s3',
    endpoint_url='http://192.168.211.133:80',
    config=botocore.client.Config(
        signature_version='s3',
        s3={'addressing_style': 'path'}
    )
)

client.create_bucket(Bucket='my-bucket')
client.put_object(Bucket='my-bucket', Key='hello.txt', Body=b'Hello!')
```

---

## Step 9 — Configure CephFS (Shared Filesystem)

CephFS provides a POSIX-compatible distributed filesystem.

### Create the filesystem volume

```bash
sudo cephadm shell -- ceph fs volume create myfs
```

This automatically:
- Creates metadata and data pools (`cephfs.myfs.meta` and `cephfs.myfs.data`)
- Deploys MDS (Metadata Server) daemons (1 active + 1 standby)

Make sure to apply the correct CRUSH rule:

```bash
sudo cephadm shell -- ceph osd pool set cephfs.myfs.meta crush_rule single-node
sudo cephadm shell -- ceph osd pool set cephfs.myfs.data crush_rule single-node
sudo cephadm shell -- ceph osd pool set cephfs.myfs.meta size 2
sudo cephadm shell -- ceph osd pool set cephfs.myfs.data size 2
```

### Mount and test

```bash
# Mount via FUSE
sudo cephadm shell -- bash -c '
  mkdir -p /mnt/myfs
  ceph-fuse /mnt/myfs
  echo "Hello via CephFS!" > /mnt/myfs/hello.txt
  cat /mnt/myfs/hello.txt
  df -h /mnt/myfs
'
```

With 60 GiB of raw storage and replication of 2, you get about **29 GiB usable**. CephFS supports all standard POSIX semantics — permissions, hard links, rename, etc.

---

## Cluster Status (Final)

```
  cluster:
    id:     5b53032c-78de-11f1-a3bb-000c29f36aa9
    health: HEALTH_WARN
            (1 pool(s) do not have an application enabled — harmless)

  services:
    mon: 1 daemons, quorum ceph-node1
    mgr: ceph-node1(active)
    mds: 1/1 daemons up, 1 standby
    osd: 3 osds: 3 up, 3 in
    rgw: 2 daemons active (1 hosts, 1 zones)

  data:
    volumes: 1/1 healthy
    pools:   9 pools, 71 pgs
    objects: 249 objects, 456 KiB
    usage:   103 MiB used, 60 GiB / 60 GiB avail
    pgs:     71 active+clean
```

A fully functional cluster with RBD, S3 object storage, and a POSIX filesystem — all on one node.

---

## What "Production" Means Here

I'm going to be straight with you: a single-node Ceph cluster is **not** what the community means by "production-ready." Ceph's core value proposition is redundancy — automatic rebalancing, no single point of failure, self-healing. A single node gives you none of that at the host level.

**What it *is* good for:**

- Development and testing
- Learning Ceph operations (OSD replacement, pool management, RGW setup)
- CI/CD integration testing
- Demos and workshops

**What it *isn't* good for:**

- Storing irreplaceable production data
- Meeting any SLA or uptime requirement
- Replacing a proper 3+ node cluster

If you're building toward a real deployment, this single-node setup is a great sandbox. Practice OSD replacement. Experiment with erasure coding. Break things and fix them. Then when you're confident, spin up your three real nodes.

---

## What We Didn't Cover

- **Multi-node expansion:** Adding nodes is just `ceph orch host add node2` + `ceph orch apply osd`. The CRUSH rule can be reverted to `host` once you have real host-level redundancy.
- **Erasure coding:** For cold storage pools, EC is more space-efficient than replication. With 3 OSDs, a `k=2,m=1` EC profile gives 33% overhead vs 200% for replication.
- **NFS-Ganesha:** CephFS can be exported over NFS for legacy clients.
- **Multi-site RGW:** For S3 geo-redundancy across data centers.

---

*Questions? Corrections? Found a better way to handle the single-node CRUSH rule? Let me know.*
