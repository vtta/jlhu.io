+++
title = "Rebalancer Logic"
date = "2023-11-29"
+++

Currently we expose memory statistics via balloon.
The statistics come in every second.

## Possible Problems
- How to facilitate burst allocations?
- How are the end performance results derived?
    - What are the performance metrics?
    - How should we collect these metrics?
    - Are they usable for comparisons?
- What happen if balloon too much or too less?

## Design: Prediction with feedback
We can process the incoming statistics in a sliding-window manner.
A prediction            


## Requirements for workloads
- Able to show realtime performance metrics
