## Paper title: Are You Sure You Want TLBs No More? --Disaggregating Tiered Memory for Virtualized Clouds with HyperTier

---

### OSDI25 results

Thank you for submitting to OSDI 2025! The reviewers acknowledged the implementation effort in this work.

Ultimately, the paper was rejected due to concerns about incremental novelty, lack of deeper motivation for the design choices, and evaluation gaps (missing hypervisor-based comparisons). In future iterations, the reviewers recommend to further distill and emphasize the novel contributions of the approach, and including a clear threat model. The reviewers also suggest strengthening the novelty claims of the PEBS based hotness tracking, and enhancing evaluation with broader benchmarks and latency analysis.

| Where | What                               | `tlb_flush` | `remote_tlb_flush` | `tlb_local_flush_one` | `tlb_local_flush_all` | `tlb_remote_flush` | Elapsed  |
| ----- | ---------------------------------- | ----------- | ------------------ | --------------------- | --------------------- | ------------------ | -------- |
| Guest | TPP                                | 1240        | 0                  | 5,587,867             | 29,658                | 15,410,466         | 6:11.85  |
| Host  | TPP                                | 89,361,453  | 21,220,056         | 5,889,040             | 22,998                | 1,205,535          | 32:37.02 |
|       | None                               | 26,402      | 0                  | 5,669,863             | 23,846                | 1,179,579          | 4:59.50  |
| Guest | PML                                | 139,373     | 491                | 5606757               | 26806                 | 1180724            | 5:39.07  |
| Guest | Ours                               | 1888        | 0                  |                       |                       |                    | 5:20.25  |
|       | (2025-03-28T01:53:31.435434+08:00) |             |                    |                       |                       |                    |          |

---

核心insight: TLB flush

- why care about it? memory virtualization -> 2D translation
- why host-based not ok?
  - what are we referring to? HeteroVisor/HeteroOS; TPP; vTMM
    - HeteroVisor/HeteroOS: direct pass due to trapping; need INVEPT for every  pagetable modification
    - TPP and all a-bit based tracking method: excessive use of INVEPT
    - vTMM: use PML to find out modified page table pages
  - detailed analysis
    - Theory: host based TLB flushing is inherently more expensive: especially INVEPT >> INVVPID > INVLPG > INVPCID
    - To detect all modifications to a page INVEPT must be used. A page might be mapped to multiple gVA with multiple A-bits. INVVPID/INVLPG/INVPCID can only flush a single mapping for a gVA. Host lack guest's rmap structure to find all those mappings.

- background needed
  - basic workflow of TMM
  - 2d pagetable
  - tlb caching and flushing






---

### Abstract

Tiered Memory technologies, including Compute Express Link (CXL.mem) and Persistent Memory (PMEM), have become primary means for memory expansion in data centers, enabling cost-effective scaling while maintaining performance by combining high-speed DRAM with larger, more affordable memory tiers. However, hypervisor system support is lagging behind, hindering their availability to the generic public through virtual machine services.

The core insight of this paper is discovering that existing hypervisor and host-based methods for managing tiered memory in virtualized environments incur significant overhead, prohibiting real-world adoption. This overhead primarily stems from the destructive TLB flushes. Virtualized environments especially rely on TLBs to hide the severe costs of two-dimensional page table walks, which require up to 5×5 page table lookups for each memory access.

To address this, we present HyperTier, both a principled and concrete design that fully decouples tiered memory provision and management in virtualized environments. In the host, we design HyperFlex for tiered memory provisioning, which also supports converting existing host-based solutions into virtualization-ready alternatives. The guest handles the complete process of hotness tracking and data placement management. We design the HyperPlace guest component, which optimizes all stages of guest tiered memory management to reduce wasted cycles and convert them into rentable CPU resources.

We evaluate all transformed and our optimized designs across two tiered memory configurations: DRAM paired with either real Persistent Memory or emulated CXL.mem. Results show that our conversion alone yields 74% performance improvement for the existing best-performing transformed design. Furthermore, our optimized design boosts performance by an additional 90% on real-world workloads running concurrently in multiple virtual machines compared to the second-best alternative.

### Intro

- Macro background: Memory expansion
- Motivation: Virtualization rely on TLB to hide translation overhead however hotness tracking in SOTA hypervisor-based TM incurs frequent tlbflushes
  - virtualization is at the heart of mordern virtualized cloud which relies on two demensional address translation
  - Nowadays uses EPT + TLB to translate and hide overhead instead of traditional software trapping
  - Access tracking exploiting pagetables requires frequenty tlb flushes
  - A ready to use solution would be managing TM in the host using SOTA or use purposely built solution
  - 
- Contribution: Responsibility disaggregation + optimized guest hotness management

### Background

- TMM
- Address translation under virtualization, TLB, flushes.
  - Software-based virtualization: HeteroVisor/HeteroOS
  - Hardware virtualization: vTMM; KVM+TPP
- SOTA: A-bit tracking
  - 

### Motivation

- 

### Design

### Evaluation





