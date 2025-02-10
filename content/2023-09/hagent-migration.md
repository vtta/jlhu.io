+++
title = "HAgent Migration"
date = "2023-09-25"
[extra]
add_toc = true
+++

Currently HAgent can distinguish between hot pages and cold pages,
but cannot tell whether they are hot/cold DRAM or PMEM pages.
In the beginning, the decision to use load latency event for access sampling
has included the consideration for media classification.
But the reality is that DRAM and PMEM access latencies are not a simple number.
To figure out whether this is a feasible solution,
we need to compare the latency distribution and figure out a way to distinguish
whether a page is placed on DRAM or PMEM without additional tracking data structures.


