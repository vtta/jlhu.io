+++
+++

### Clarifications

In this version, we further distilled our novelty into the princile of guest-delegation, provided motivation for such delegation through both theoritical analysis of virtualization and empirical evlauation of eixsting hypervisor-based approaches.

We also included evaluation on hypervisor-based approaches and latency sensitive real world applicaiotn in our paper.

### Origin review summary

Ultimately, the paper was rejected due to concerns about **incremental novelty**, lack of deeper motivation for the design choices, and evaluation gaps (missing hypervisor-based comparisons). In future iterations, the reviewers recommend to further distill and emphasize the novel contributions of the approach, and including a clear threat model. The reviewers also suggest strengthening the novelty claims of the PEBS based hotness tracking, and enhancing evaluation with broader benchmarks and latency analysis.

### v1

## Clarifications on Significant Changes from Previous Version

In response to reviewer feedback about incremental novelty and design motivation, we have made the following substantial improvements:

1. Unified Theoretical Framework
    We have fully developed the "guest-delegation" principle as a cohesive theoretical framework underlying our approach, clarifying how this represents a paradigm shift rather than an incremental improvement over existing hyperivsor-based and kernel-based solutions.

2. Strengthened Motivation

   We have added detailed empirical analysis in Section 2.3 demonstrating the fundamental limitations of hypervisor-based approaches, including:

   - Quantitative measurement of TLB flushes and overhead (Tbl. 1)
   - Analysis of EPT-friendly PEBS scalability issues (Fig. 2)

3. Expanded Evaluation

   We have addressed key evaluation gaps by:

   - Adding direct comparisons with hypervisor-based approaches (TPP-H) across seven real-world workloads
   - Including latency analysis for interactive applications (Fig. 12), showing 23% reduction in tail latency

4. Clarified Novelty of PEBS Approach

   We have substantiated our claims about EPT-friendly PEBS with:

   - Technical explanation of previous architectural limitations (Section 2.3.2)
   - Analysis of isolation properties ensuring multi-tenant security
   - Context-switch-based sampling implementation that reduces CPU overhead by 16×

5. Added Threat Model Discussion
   Despite this is not the main focus of our paper, we have included a threat model in Section 7.2, addressing security considerations for untrusted guest code running in multi-tenant environments.

