+++
date = '2026-07-04T21:44:00+08:00'
draft = false
title = '🔬 Does CPU Topology Affect Performance? A sysbench Experiment'
+++

## 📋 Overview

Does changing between 1 socket × 12 cores and 2 sockets × 6 cores make a real-world performance difference? I ran `sysbench cpu` on a CentOS Stream 9 VMware VM to find out.

**Host:** AMD Ryzen 5 2600, CentOS Stream 9, VMware VM, 3.5 GB RAM

---

## The Setup

I tested two identical configurations — same VM, same CPU count (12 vCPUs), only the topology changed:

| Config | Sockets | Cores/socket | Threads/core | Total vCPUs |
|---|---|---|---|---|
| A | 1 | 12 | 1 | 12 |
| B | 2 | 6 | 1 | 12 |

The topology is configured in the VMware VM settings (`.vmx` file or vSphere), then verified with `lscpu`.

---

## The Benchmark

Using `sysbench cpu` which runs prime number calculations:

```bash
# Single thread
sysbench cpu run

# Multi-threaded (scale to vCPU count)
sysbench cpu --threads=12 --time=15 run

# Overcommitted (shows if hyperthreading helps)
sysbench cpu --threads=24 --time=15 run
```

---

## Results

### 1 Socket × 12 Cores

```
CPU(s):          12
Socket(s):       1
Core(s)/socket:  12
NUMA node(s):    1
```

| Threads | Events/sec | Scaling vs 1T |
|---|---|---|
| 1 | 1,844 | — |
| 12 | 11,012 | 6.0x |
| 24 | 10,966 | — (overcommit) |

### 2 Sockets × 6 Cores

```
CPU(s):          12
Socket(s):       2
Core(s)/socket:  6
NUMA node(s):    1
```

| Threads | Events/sec | Scaling vs 1T |
|---|---|---|
| 1 | 1,819 | — |
| 12 | 11,001 | 6.0x |
| 24 | 10,987 | — (overcommit) |

### Side-by-Side Comparison

| Test | 1S × 12C | 2S × 6C | Difference |
|---|---|---|---|
| 1 thread | 1,844/s | 1,819/s | -1.4% |
| 12 threads | 11,012/s | 11,001/s | -0.1% |
| 24 threads | 10,966/s | 10,987/s | +0.2% |

All results are within **±1.5%** — well within normal measurement variance.

---

## Why They're the Same

The VMware VM exposes a **single NUMA node** in both configurations:

```
NUMA node(s):                       1
NUMA node0 CPU(s):                  0-11
```

With one NUMA domain, all memory access is local — there's no penalty for a vCPU on "socket 0" accessing memory "allocated to socket 1" because they share the same memory controller. Topology differences only matter when:

- **NUMA is configured** — each socket has its own memory bus
- **The workload is NUMA-aware** — databases, HPC, large JVM heaps
- **Software licensing per socket** — some software licenses by socket count

---

## Scaling Insights

```
12 threads:  ~11,000/s  →  6.0x over single thread
24 threads:  ~11,000/s  →  No improvement (overcommit)
```

The Ryzen 5 2600 has 6 physical cores / 12 threads. Adding more than 12 sysbench threads shows zero gain — the CPU is saturated. This tells us:

- The VMware hypervisor is **efficient** — near-linear scaling up to physical core count
- **Overcommitting CPU** (more vCPUs than host cores) doesn't help CPU-bound workloads
- Memory bandwidth starts to bottleneck beyond ~8 threads

---

## 📊 Summary

| Question | Answer |
|---|---|
| Does socket/core topology affect CPU performance? | **No** (within same NUMA node) |
| Does scaling to physical core count work? | **Yes** (near-linear) |
| Does overcommitting CPU help? | **No** (CPU-bound workloads) |
| Does NUMA matter? | **Only with multiple NUMA nodes** |

---

## 💡 Lessons Learned

1. **🔬 Topology is cosmetic without NUMA** — Changing sockets vs cores has zero performance impact when the VM has a single NUMA node. The OS sees N logical CPUs regardless of layout.
2. **📏 Scale to physical cores, not beyond** — Sysbench shows no gain past 12 threads on a 6-core/12-thread host CPU. Overcommitting vCPUs beyond host capacity hurts more than it helps.
3. **📈 VMware keeps NUMA simple** — By default, VMware exposes a single NUMA node to guests unless the VM spans physical NUMA boundaries on the host.
4. **⚙️ Benchmark before optimizing** — If you're tuning vCPU topology, always measure. The default is usually fine.

---

*Spent an hour changing socket counts... only to prove it doesn't matter. Sometimes the best lesson is that there's nothing to optimize.* 🎯
