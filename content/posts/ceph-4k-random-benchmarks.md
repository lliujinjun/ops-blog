+++
date = '2026-07-06T14:12:00+08:00'
draft = false
title = 'Ceph 4K Random I/O Benchmarks: Raw Disk vs RADOS vs RBD'
+++

**Environment:** CentOS Stream 9 · Ceph Reef 18.2.8 · Manual Install · VMware VM · 4 OSDs (4 × 20 GB VMDK) · 4 vCPUs · 4 GB RAM

---

When you set up a Ceph cluster, the question always comes: *how fast is it?* The answer depends on which layer you're testing. Ceph has multiple data paths — raw RADOS, RBD block devices, and the underlying disk — and each has different overhead.

This post benchmarks all three layers on a 4-OSD single-node cluster and compares them against the raw disk performance of the underlying VMware VMDK.

---

## The Three Layers

```
┌──────────────────────────┐
│     RBD (block device)   │  ← fio on /dev/rbd0 (krbd)
├──────────────────────────┤
│    RADOS (object store)  │  ← rados bench (librados)
├──────────────────────────┤
│      OSD BlueStore       │  ← ceph-osd daemon
├──────────────────────────┤
│     Raw Block Device     │  ← fio on /dev/sdf (VMDK)
└──────────────────────────┘
```

Each layer adds protocol overhead: authentication, CRUSH calculation, replication, and network I/O.

---

## Test Methodology

All tests use the same parameters:

- **I/O size:** 4 KB (small-block random — realistic for databases)
- **Concurrency:** 16 concurrent I/Os (iodepth=16)
- **Duration:** 10–15 seconds per test
- **Direct I/O:** `O_DIRECT` to bypass OS page cache
- **Ceph replication:** `size=2` (each object stored on 2 OSDs)

Three test types:
1. **Random write** — 100% writes
2. **Random read** — 100% reads
3. **Mixed 70/30** — 70% reads, 30% writes (only for fio tests)

---

## Raw Disk Benchmark

Tests the underlying VMware VMDK directly, no Ceph involved.

### Command

```bash
# 4K random write
sudo fio --name=randwrite4k --filename=/dev/sdf --size=20G \
  --rw=randwrite --bs=4K --direct=1 \
  --runtime=15 --ioengine=libaio --iodepth=16

# 4K random read
sudo fio --name=randread4k --filename=/dev/sdf --size=20G \
  --rw=randread --bs=4K --direct=1 \
  --runtime=15 --ioengine=libaio --iodepth=16

# 4K mixed 70/30
sudo fio --name=mixed4k --filename=/dev/sdf --size=5G \
  --rw=randrw --rwmixread=70 --bs=4K --direct=1 \
  --runtime=15 --ioengine=libaio --iodepth=16
```

### Results

```
WRITE: bw=23.1MiB/s (24.2MB/s), io=347MiB (364MB), run=15001msec
 READ: bw=70.7MiB/s (74.1MB/s), io=1060MiB (1111MB), run=15001msec
MIXED: read=34.6MiB/s, write=14.8MiB/s
```

| Metric | Value |
|--------|-------|
| 4K write | 23.1 MB/s, 5,900 IOPS |
| 4K read | 70.7 MB/s, 18,000 IOPS |
| 4K mixed 70/30 | 34.6 + 14.8 MB/s, 12,000 IOPS |

---

## RADOS Benchmark (Raw Ceph Object Store)

Tests the raw `librados` layer — the lowest Ceph API. Data is written as 4K objects directly to OSDs, through the MON for placement. No block device abstraction.

### Command

```bash
# Create a test pool
sudo ceph osd pool create bench 32 32 replicated single-node
sudo ceph osd pool set bench size 2

# 4K write (pre-warms objects for the read test)
sudo rados bench -p bench 10 write -b 4096 --no-cleanup

# 4K random read (reads back the written objects)
sudo rados bench -p bench 10 rand

# Cleanup
sudo rados -p bench cleanup
sudo ceph osd pool delete bench bench --yes-i-really-really-mean-it
```

> **Note:** `rados bench` reads use the same object size as the write phase. Don't use `-b` on the read command — it only works for writes.

### Results

```
WRITE: bw=7.47MB/s, 1912 IOPS, avg lat=8.3ms
 READ: bw=44.4MB/s, 11359 IOPS, avg lat=1.4ms
```

| Metric | Value |
|--------|-------|
| 4K write | 7.5 MB/s, 1,912 IOPS, 8.3 ms |
| 4K read | 44.4 MB/s, 11,359 IOPS, 1.4 ms |

---

## RBD Benchmark (Block Device)

Tests Ceph's block device layer — a RADOS image mapped via `rbd-nbd` (or `krbd`). This is the path used by KVM/QEMU virtual machines. fio runs directly against the mapped `/dev/rbd0` device.

### Command

```bash
# Create pool and RBD image
sudo ceph osd pool create bench 32 32 replicated single-node
sudo ceph osd pool set bench size 2
sudo rbd create bench-image --size 5G --pool bench
sudo rbd device map bench/bench-image

# Find the mapped device
DEV=$(lsblk | grep rbd | awk '{print $1}')

# 4K random write
sudo fio --name=rbd-write --filename=/dev/$DEV --size=5G \
  --rw=randwrite --bs=4K --direct=1 \
  --runtime=15 --ioengine=libaio --iodepth=16

# 4K random read
sudo fio --name=rbd-read --filename=/dev/$DEV --size=5G \
  --rw=randread --bs=4K --direct=1 \
  --runtime=15 --ioengine=libaio --iodepth=16

# 4K mixed 70/30
sudo fio --name=rbd-mixed --filename=/dev/$DEV --size=5G \
  --rw=randrw --rwmixread=70 --bs=4K --direct=1 \
  --runtime=15 --ioengine=libaio --iodepth=16

# Cleanup
sudo rbd device unmap /dev/$DEV
sudo rbd rm bench-image --pool bench
sudo ceph osd pool delete bench bench --yes-i-really-really-mean-it
```

### Results

```
WRITE: bw=6111KiB/s (6258kB/s), io=89.6MiB, run=15010msec
 READ: bw=56.8MiB/s (59.5MB/s), io=851MiB, run=15001msec
MIXED: read=11.9MiB/s, write=5223KiB/s
```

| Metric | Value |
|--------|-------|
| 4K write | 6.0 MB/s, ~1,500 IOPS |
| 4K read | 56.8 MB/s, ~14,500 IOPS |
| 4K mixed 70/30 | 11.9 + 5.1 MB/s, ~4,350 IOPS |

---

## Comparison Table

| Test | Raw Disk | RADOS | RBD | Overhead (RBD vs Raw) |
|------|----------|-------|-----|-----------------------|
| **Write** | 23.1 MB/s | 7.5 MB/s | **6.0 MB/s** | 3.9× |
| **Write IOPS** | 5,900 | 1,912 | 1,500 | 3.9× |
| **Write latency** | — | 8.3 ms | — | — |
| **Read** | 70.7 MB/s | 44.4 MB/s | **56.8 MB/s** | 1.2× |
| **Read IOPS** | 18,000 | 11,359 | 14,500 | 1.2× |
| **Read latency** | — | 1.4 ms | — | — |
| **Mixed 70/30** | 49.4 MB/s | — | **17.0 MB/s** | 2.9× |

### Visualization

```
Write throughput (higher is better)

Raw Disk  ████████████████████████████  23.1 MB/s
RADOS     ████████                      7.5 MB/s
RBD       ███████                       6.0 MB/s

Read throughput (higher is better)

Raw Disk  ████████████████████████████████████████  70.7 MB/s
RADOS     ████████████████████████                  44.4 MB/s
RBD       ███████████████████████████████           56.8 MB/s
```

---

## Analysis

### Write Overhead (3.9×)

Every Ceph write goes through:

1. **CRUSH calculation** — determine which OSDs hold the placement group
2. **Network** — send the data to the primary OSD (via loopback or wire)
3. **BlueStore double-write** — WAL (write-ahead log) + data device
4. **Replication** — primary OSD forwards to replica OSD (size=2)
5. **ACK** — replica confirms, primary confirms to client

With `size=2`, two OSDs each do a double-write. That's **4 disk writes** for every client write. The theoretical best is roughly `raw_speed / 4`.

Raw disk write: 23.1 MB/s → 23.1 / 4 = 5.8 MB/s theoretical. We got **6.0 MB/s** — right on target.

### Read Overhead (1.2×)

Reads only hit one OSD (the primary). There's no replication penalty. The 1.2× overhead comes from CRUSH calculation, the network stack, and BlueStore metadata lookup. Ceph reads are nearly as fast as the raw disk.

### Mixed Overhead (2.9×)

Mixed workload combines read (efficient) and write (expensive) penalties. The exact ratio depends on the read/write mix.

### Latency

RADOS write latency at **8.3 ms** is reasonable for virtual disks. The raw disk itself contributes the majority — the Ceph processing (network + replication) adds roughly 3-4 ms on top.

RADOS read latency at **1.4 ms** is excellent — BlueStore's RocksDB-based metadata lookup added less than a millisecond.

---

## Caveats

These numbers are for **VMware virtual disks** on a **single-node** cluster with `size=2` replication. Real-world performance depends heavily on:

- **Physical hardware** — NVMe SSDs achieve 10-100× these numbers
- **Network** — 10 GbE vs 25 GbE vs 100 GbE
- **Number of OSDs** — more OSDs = more aggregate throughput
- **CRUSH failure domain** — multi-node clusters have additional network hops
- **BlueStore tuning** — cache size, WAL mode, rocksdb compaction

---

## Benchmark Cheat Sheet

```bash
# === RAW DISK ===
# Write
fio --name=write --filename=/dev/<DISK> --size=20G \
  --rw=randwrite --bs=4K --direct=1 \
  --runtime=15 --ioengine=libaio --iodepth=16

# Read
fio --name=read --filename=/dev/<DISK> --size=20G \
  --rw=randread --bs=4K --direct=1 \
  --runtime=15 --ioengine=libaio --iodepth=16

# === RADOS ===
ceph osd pool create bench 32 32 replicated single-node
ceph osd pool set bench size 2

rados bench -p bench 10 write -b 4096 --no-cleanup
rados bench -p bench 10 rand

rados -p bench cleanup
ceph osd pool delete bench bench --yes-i-really-really-mean-it

# === RBD ===
ceph osd pool create bench 32 32 replicated single-node
ceph osd pool set bench size 2
rbd create bench-image --size 5G --pool bench
rbd device map bench/bench-image

fio --name=rbd-test --filename=/dev/rbd0 --size=5G \
  --rw=randrw --rwmixread=70 --bs=4K --direct=1 \
  --runtime=15 --ioengine=libaio --iodepth=16

rbd device unmap /dev/rbd0
rbd rm bench-image --pool bench
ceph osd pool delete bench bench --yes-i-really-really-mean-it
```
