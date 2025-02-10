+++
title = "Experiment Note"
date = "2024-01-02"
# [extra]
# add_toc = true
+++

Currently, the overall performance is very bad.
1. We have already known the coverage problem of SDH,
    which is only good for identify around top 5% instead of top 30%.
2. We tried to solve this by switching GUPS access pattern to a zipf distribution.
    We found the property of only identifying top 5% does not change with workload.
    In zipf, the top 5% key should make up more than 30% total accesses.
    However, there is not a visible change to end to end performance.


## Latency analysis
Objective: 
check if the sample collection and migration mechanism process samples timely enough.

Method:
By intoducing a series of timestamps during the lifecycle of a sample,
we can calculate:
1. The collection latency between PEBS hardware generation and software collection.
2. Hotness tracking latency between collection and migration.


## CPU usage analysis

## Hotset coverage analysis
