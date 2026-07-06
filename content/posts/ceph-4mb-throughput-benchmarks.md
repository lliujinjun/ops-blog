+++
date = '2026-07-06T14:05:00+08:00'
draft = false
title = 'Ceph 4MB Throughput Benchmarks: Raw Disk vs RADOS'
+++

**Environment:** CentOS Stream 9 · Ceph Reef 18.2.8 · Manual Install · VMware VM · 4 OSDs (4 × 20 GB VMDK) · 4 vCPUs · 4 GB RAM

---

This benchmark tests large-block (4 MB) throughput — the kind of workload you'd see with video files, backups, or large object storage. The goal is to see how close Ceph RADOS gets to raw disk bandwidth for sequential and random I/O at 4 MB block sizes.

---

## Test Methodology

- **Block size:** 4 MB (large I/O)
- **Concurrency:** 16 concurrent I/Os
- **Duration:** 10 seconds per test
- **Direct I/O:** `O_DIRECT`
- **Ceph replication:** `size=2`

Three test types:
1. **Sequential write** — write new objects
2. **Random read** — read existing objects randomly (bypasses page cache for Ceph, but VMware hypervisor cache may still apply)
3. **Sequential read** — read existing objects sequentially

---

## Raw Disk Benchmark

Tests `/dev/sdf` (a raw 20 GB VMDK) directly via `fio`.

### Command

```bash
# Sequential write
sudo fio --name=write --filename=/dev/sdf --size=20G \
  --rw=write --bs=4M --direct=1 \
  --runtime=10 --ioengine=libaio --iodepth=16

# Random read
sudo fio --name=randread --filename=/dev/sdf --size=20G \
  --rw=randread --bs=4M --direct=1 \
  --runtime=10 --ioengine=libaio --iodepth=16

# Sequential read
sudo fio --name=seqread --filename=/dev/sdf --size=20G \
  --rw=read --bs=4M --direct=1 \
  --runtime=10 --ioengine=libaio --iodepth=16
```

### Results

```
WRITE: bw=209MiB/s (220MB/s), 2156MiB/10295msec
RANDREAD: bw=3855MiB/s (4043MB/s), 20.0GiB/5312msec
SEOREAD: bw=3791MiB/s (3975MB/s), 20.0GiB/5402msec
```

| Test | Bandwidth | IOPS | Duration |
|------|-----------|------|----------|
| Sequential write | **220 MB/s** | 52 | 10.3 s |
| Random read | **4,043 MB/s** | 963 | 5.3 s |
| Sequential read | **3,975 MB/s** | 947 | 5.4 s |

> **Caveat on read numbers:** The random and sequential read tests completed the full 20 GB disk in ~5 seconds at ~4 GB/s. This is **not** the physical disk speed — it's VMware's hypervisor block cache serving the data. A warm cache on a virtual disk will always deliver unrealistic numbers for large-block reads. The write test is the reliable metric here because `O_DIRECT` writes must hit the physical storage, and 220 MB/s is realistic for a single VMDK on a shared VMware datastore.

---

## RADOS Benchmark

Tests the raw `librados` layer with the default 4 MB object size.

### Command

```bash
# Create pool
sudo ceph osd pool create bench 32 32 replicated single-node
sudo ceph osd pool set bench size 2

# Sequential write
sudo rados bench -p bench 10 write --no-cleanup

# Random read (reads back written objects)
sudo rados bench -p bench 10 rand

# Sequential read
sudo rados bench -p bench 10 seq

# Cleanup
sudo rados -p bench cleanup
sudo ceph osd pool delete bench bench --yes-i-really-really-mean-it
```

### Results

```
WRITE: bw=103 MB/s, 25 IOPS, avg lat=621ms
RANDREAD: bw=262 MB/s, 65 IOPS, avg lat=241ms
SEQREAD: bw=557 MB/s, 139 IOPS, avg lat=109ms
```

| Test | Bandwidth | IOPS | Avg Latency |
|------|-----------|------|-------------|
| Sequential write | **103 MB/s** | 25 | 621 ms |
| Random read | **262 MB/s** | 65 | 241 ms |
| Sequential read | **557 MB/s** | 139 | 109 ms |

---

## Comparison

| Test | Raw Disk | RADOS (Ceph) | Overhead |
|------|----------|--------------|----------|
| **Sequential write** | 220 MB/s | **103 MB/s** | **2.1×** |
| **Random read** | 4,043 MB/s | **262 MB/s** | **15.4×** |
| **Sequential read** | 3,975 MB/s | **557 MB/s** | **7.1×** |

### Write Overhead (2.1×) — The Realistic Metric

The sequential write test is the most reliable comparison. Raw disk: 220 MB/s. Ceph RADOS: 103 MB/s.

The overhead comes from:
- **Replication ×2** — `size=2` means every write goes to two OSDs, cutting theoretical throughput in half
- **BlueStore double-write** — WAL (write-ahead log) + data device adds ~10-20% overhead
- **Network stack + CRUSH** — RADOS protocol processing

```
Raw:  220 MB/s × 1 OSD × 1 write = 220 MB/s
Ceph: 220 MB/s × 2 OSDs × 2 writes (BlueStore) / 4 OSDs = ~110 MB/s theoretical
      Actual: 103 MB/s — close to theoretical limit
```

### Read Overhead — VMware Cache Distortion

The raw disk read numbers (4,043 MB/s and 3,975 MB/s) are **not physical disk speeds**. They're VMware's block cache serving data from RAM. This is why the Ceph read numbers look dramatically slower by comparison — Ceph reads go through BlueStore on each OSD, which does actual disk I/O.

**Ceph's actual sequential read (557 MB/s)** is the more meaningful number. It represents real disk-to-client throughput across 4 OSDs:

```
557 MB/s / 4 OSDs = ~139 MB/s per OSD
```

That's a healthy sequential read rate for a virtual SCSI disk on VMware.

---

## Summary

| What this measures | Result |
|--------------------|--------|
| Raw single-disk write throughput | 220 MB/s |
| Ceph 4-OSD write throughput (size=2) | 103 MB/s |
| Ceph write overhead factor | **2.1×** (replication + BlueStore) |
| Ceph sequential read (aggregate) | 557 MB/s |
| Ceph random read (aggregate) | 262 MB/s |

For large-block workloads (video, backups, bulk storage), Ceph delivers roughly **half the raw disk throughput** per OSD after accounting for replication. The trade-off — redundancy, self-healing, and scalability — is worth the performance cost for most use cases.
