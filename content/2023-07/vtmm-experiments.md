+++
title = "vTMM Experiments"
date = "2023-07-31"
# [extra]
# add_toc = true
+++

## Environment setup
- Software: Ubuntu 18.04 + Linux 5.4.142 + QEMU 3.1.0
- Host CPU: 2x 2.2GHz 24C/48T Cascade Lake-SP
- VM CPU: 8 or 16 vCPU
- VM memory:
    1. DRAM+NVM: 32G + 192G (local PMEM)
    2. ~~DRAM+CXL: 32G + 192G (remote DRAM)~~ (results omitted)

## Section Structure
|      | title                                | columns |
| ---- | ------------------------------------ | ------- |
| 1    | experimental setup                   |         |
| 2    | experimental approach                | 3/4     |
| 3    | page tracking efficiency             | 2+1/2   |
| 4    | page migration efficiency            | 1+3/4   |
| 5    | ablation study (page classification) | 1+1/4   |
| 6    | fixed/dynamic hot set                | 1       |
| 7    | sensitivity to dram size             | 1       |
| 8    | thp support                          | 1+3/4   |
| 9    | multi-vm co-running                  | 2+1/2   |

- Page Tracking
    - Overhead
    - Accuracy (4.3)
- Page Classification
    - vs LRU (4.8)
- Page Migration
    - Migration Speed
    - VM Slowdown
    - vs Linix/HeMem (4.4)
- Rebalancing
    - (4.9) (how? baseline?)
- Macro
    - vs AutoNUMA/Intel MM/Nimble

In summary, the structure is as expected. It contains micro and macro benchmarks.
Macro benchmarks demonstrate overall system performance and micro benchmarks shows performance of every design decision.
The demonstrated designs covers collection, identification, migration and rebalancing.
In particular, it contains more details on collection and rebalancing.

## Tracking
- Environment: a single warmed 64G VM
- Workload:
    - Accuracu: random access over 10G memory
    - Overhead: SPEC.CPU.2006.429 graph500.bfs graph500.sssp redis.ycsb-custom (50r/50u; 4K KV; hotspot; 80% hit 20% data)

To demonstrate the accuracy of vTMM, the authors compared EPT and GPT scanning.
The results strongly favours vTMM, because page table scanner scans mapped page,
even when guest pages are not used, they are still mapped in the host.

This Benchmark will favour ours even more, because we are a sample based approach,
the scanned memory will only contains captured pages.

**Problems:**
- **How is accuracy demonstrated?**
- What is the relation between scanned pages and actually accessed/hot pages?
- Will a page be counted twice?
- Why not compare with page scanner in the guest?

