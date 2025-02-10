+++
title = "Review: Aceso: Achieving Efficient Fault Tolerance in Memory-Disaggregated Key-Value Store"
date = "2023-10-17"
[extra]
add_toc = true
+++

## Paper summary


## Details

### Abstract 
Replication-based fault tolerance technique incurs:
1. Excessive memory space
2. Costly synchronization

Solution: *XOR-based erasure coding* and *first replication then XORing*
1. Index:
    1. async replication: reduce synchronization
    2. versioning: recovery
2. KV data: XOR: save memory space

Problems:
1. Novelty is not clear.
    1. How is fault tolerance different in DM compared to traditional distributed systems?
        What are the unique challenges?
    2. What are the existing solutions in ensuring fault tolerance in DM?
        Is replication the only one?
    3. Is erasure coding another kind of replication with finer granularity?
2. Writing is not clear
    1. What is the relation between consistency and fault tolerance?
        Are we comparing different fault-tolerance techniques under the same level of consistency guarantee?
3. According to CAP theorem, are you trading availability for consistency and fault-tolerance (i.e. CP databases)?


For the background:



## Misc
- Why should we care about memory efficiency?
    Isn't it the goal of DM?

