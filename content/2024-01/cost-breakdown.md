+++
title = "Cost Breakdown"
date = "2024-01-10"
# [extra]
# add_toc = true
+++

To find out the root cause of our bad performance,
we should be able to quantify the slowdown introduced by every piece of the system.

## Performance indicator
Memory intensive benchmark: gups

## Breakdown
- Intra-vm
    - Data path
        - Slow memory hit
    - Control path
        - PEBS
            - Hardware sample generation (extra memory write)
            - Software sample collection (extra memory read)
            - Perf subsystem cost
        - Heterogeneous management
            - Hotness tracking
            - Migration
- Outside:
    - VM
        - `vmexit/vmentry`
    - Haredware
        - TLB 
        - Cache


## Slow memory hit
We cannot derived the correct slow memory hit rate from PEBS samples due to the sampling nature.
To achieve that, we can configure two additional counting events.


## Todo
- [ ] Profile CPU time of sample collection, hotness identification and page migration.
- [ ] Profile the amount of data migrated and averaged memory bandwidth used.
- [ ] Profile `vmexit` and TLB-miss involved.
- [ ] Make `memtis` work with our testing infrastructure.

