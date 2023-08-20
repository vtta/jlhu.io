+++
title = "Review: [ASP-DAC '24] CHIME: Concurrent Hierarchical In-Memory Processing using STT-RAMs"
date = "2023-08-20"
# [extra]
# add_toc = true
+++

### What is the problem this paper trying to solve?
Unlinke traditional PiM (processing in memory),
this paper consider processing in each level of the memory hierarchy,
i.e. L1, L2 cache (PiC) and main memory.

The reason for doing so includes performance/energy/data movement/silicon size concerns.

### Why focusing on STT-RAMs?
Data corruption and silicon size.

PiC/PiM performs logical AND/OR/XOR on bits using
concurrent activation of sub-arrays sharing the same bit-lane.
Concurrent activation in SRAM might cause data corruption
which introduce a silicon area overhead to facilitate this.
For DRAM, additional energy is needed to perform refresh
before any computation.

However, STT-RAMs have none of these shortcommings.
But STT-RAMs are slower, which addresses the need for PiC.

### Motivation for hierarchical PiM?
Simple logical operations in traditional PiM/PiC have limited applicabllity to real world application.
But introducing shifting and arithmatic bring huge latency slowdown (~2.7x).
Concurrently utilizing multiple PiC/PiM CUs might reduce such overheads.


### How does the CU grouping save data transferred?
As shown in the pipline figure (i.e. figure 3d),
if two consecutive instructions used two CU (e.g. add and then multiply),
data transfer is needed between memory hierarchy.
If we group the two CUs and put it to one level,
the data transfer can be saved.

The authors proposes to inspect all instruction pairs in a workload application,
and find out those most occuring ones.
Then they assign the CU pair for each stage according to mapping strategy.

## Summary
This paper considers a novel hierarchical PiC+PiM architecture
with compute units residing in each cache/memory level and forming an overall pipelined structure.
STT-RAM and its relaxed retention variant are used in every level to achieve energy saving.
The CUs in each level are furthur optimized to only support
the most occuring instructions from worload applications.
The idea is novel, and the results are substantial.
The limitations of the paper include its application tailered CUs
and the huge area overhead if supporting all complex CUs in the mean time.


## Comments

- ~~The authors use relaxed retention STT-RAM for caches
    which needs [refresh](10.1109/HPCA.2011.5749716) to avoid data loss.
    It's unclear how the refresh problem is addressed and
    what are the energy consumption results.~~ Figure 3d.
- It's unclear whether the performance gain in pipelining is worth the
    latency and energy overhead spent transferring data between stages/hierarchy.
- It's unclear how the pipelined CU solves data dependencies.
- It's unclear how the workload awareness CUs impacts practicality.
- It's unclear whether instruction recording is needed during compilation or runtime,
    and if so, how the correctness of execution results is guaranteed.
- It's unclear whether the overall area overhead is the sum of every CU group's overhead.
- ~~It's unclear how can expensive to write STT-RAM have better energy efficiency.~~
    Shown in prior work (citation 2).
