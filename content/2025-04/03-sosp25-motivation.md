+++

+++

## Motivation

- TLB flush
  - Analysis of SOTA
    - why use H-TPP
      - No opensource 
  - Demo by experiments 
    - Guest TPP vs Host TPP
  - Explanation 
    - H-TPP: EPT -> invept
    - HeteroVisor/HeteroOS/vTMM: despite tracking GPT
      - two kinds of flush instructions, full flush and gVA flush.
        INVEPT >> INVVPID / INVLPG / INVPCID
      - GPT PTE cannot be reversed back to gVA
      - Hypervisor-based management requires full flush 
  
- EPT-friendly PEBS
  - Feasibility

- Scalability issue of naive guests
  - Delicated Polling 

#### version 1

```
= Motivation

// In this section, we identify three key challenges that current approaches face: TLB flush overhead, PEBS accessibility, and scalability concerns. These challenges inform our guest-delegated design principle.

== TLB Flush Overhead <motivation-overhead>

Access tracking in existing hypervisor-based tiered memory management solutions depends on TLB flush-intensive PTE.A bits for hotness information.
To quantify this overhead, we compared TLB flush instruction counts between hypervisor-based and guest-based solutions.

Due to the lack of available source code and omitted design details in existing hypervisor-based solutions @vee15heterovisor @socc16raminate @isca17heteroos @eurosys23vtmm, we converted the host-based TPP @asplos23tpp to a hypervisor-based solution (H-TPP) by integrating its PTE.A scanning backend with KVM's MMU notifier. 
This allows it to observe guest accesses through PTE.A bits in EPT.
For guest-based solutions, we evaluated a direct application of the host-based TPP in guests (G-TPP).

For our evaluation, we limited the hypervisor-based system to 36 GB DRAM during boot, while for guest-based solutions, we started a VM with 36 GB DRAM.
We used PMEM as the SMEM tier and ran a GUPS workload with a 144 GB total footprint using 36 threads, as shown in fig-motivation-tlb-flush.
The results reveal that the hypervisor-based solution H-TPP generates 4.7× TLB flush instructions compared to G-TPP, resulting in 2.5× total execution time.

The severe performance penalty stems from the necessity of destructive full invalidation of all EPT mappings.
TLB flush instructions fall into two categories: full invalidation (`invept`) and single gVA invalidation (`invvpid`/`invpcid`/`invlpg`).
Hypervisor-based solutions, which capture GPT entries through faulting @vee15heterovisor @isca17heteroos or scanning @eurosys23vtmm, can only access GPT and EPT entries containing gPA and hPA.
Without gVA information, they must resort to full invalidation to ensure capturing future PTE.A/D bits.
In contrast, guest-based solutions can follow the entire GPT walk process and extract the initial gVA, enabling the use of more efficient single-address invalidations.
Furthermore, guest-based solutions can leverage EPT-friendly PEBS instead of TLB flush-intensive PTE.A/D bits.

== EPT-Friendly PEBS

Contrary to prior assumptions @eurosys23vtmm @osdi24memstrata, we find that PEBS sampling is now well-supported with strong isolation for guest states under virtualization @lkml22eptfriendlypebs.
Cross-architecture support for PMU sampling also exists @lkml24riscvguestsampling, allowing PEBS-based hotness classification to extend to a wide range of cloud machines with simple modifications to the collected events.

The primary concern regarding guest PEBS support has been the potential breach of isolation boundaries caused by sharing the sample buffer, potentially leaking sensitive information across VMs.
Prior systems intuitively assumed that PEBS enabled in guests would generate samples and write to the host OS's PEBS buffer @eurosys23vtmm, leaking load/store addresses via generated samples.
However, we found this is not the case.

The PEBS sample buffer is part of the CPU debug control data structure and is managed via a debug control register.
Hardware virtualization automatically switches to a guest-private PEBS buffer through the `vmcs.debugctl` field in the Virtual Machine Control Structure.
Samples generated while executing different VMs are written directly to their private buffers, ensuring proper isolation.

== Scalability Challenges

With PEBS now available in guests, it might seem intuitive to apply existing PEBS-based designs directly within guest VMs.
However, cloud environments often run numerous VMs concurrently on a single machine, which would compound management overhead if existing designs were directly repurposed.
For instance, HeMem @sosp21hemem dedicates one CPU core to pulling samples from the PEBS buffer, which would waste one core per VM, a prohibitive overhead in multi-tenant environments.

We conducted a scalability study of existing designs using both PTE.A/D-based (TPP) and PEBS-based (Memtis) approaches, as shown in @fig-motivation-scalability.
We launched up to nine virtual machines on a 36-core system and ran 8.1 billion GUPS transactions with a 126 GiB working set divided evenly across all VMs while preserving the access distribution.

The results demonstrate that naively using TPP in guests could waste more than 3.5 CPU cores in a 36-core system, while even the optimized PEBS design of Memtis still wastes approximately half a core.
In cloud environments where CPU resources are rented to customers with the goal of maximizing utilization, such wastage would increase total cost of ownership (TCO), potentially negating the benefits of memory expansion through tiered memory.

// These findings highlight the need for a fundamentally new approach to tiered memory management in virtualized environments—one that leverages the benefits of guest-side access tracking while addressing the scalability challenges of existing designs. This motivates our guest-delegated principle and the development of HyperFlex and HyperPlace, which we detail in the following sections.
```



#### Draft

```
= Motivation

== TLB flush
Access tracking in existing hypervisor-based tiered memory management solutions, depends on TLB flush intensive PTE.A bits hotness information.
We present a TLB flush instruction count measurement comparison between hypervisor-based and guest-based solutions.
Because the lack of avaialble source code and ommitance of design details of existing hypervisor-based solutioins @vee15heterovisor @socc16raminate @isca17heteroos @eurosys23vtmm, we convert host-based TPP @asplos23tpp to a hypervisor-based soltion, named H-TPP, by plugging its PTE.A scanning backend with KVM's MMU notifier, allowing it to observe guest accesses through PTE.A bits in EPT.
For guest-based solutions, we present results of directly applying existing host-based solution TPP in guests, named G-TPP, and our HyperTier design.

For hypervisor-based, we limit usable DRAM of the evaluation machine to 36G during boot, for guest-based, we start a VM with 36G DRAM.
We use PMEM as the SMEM tier.
We run a GUPS workload with 144G total footprint with 36 threads and plot results in @fig-motivation-tlb-flush.
The results show hypervisor-based solution H-TPP exhibits 4.7x total TLB flush instruction counts compared to G-TPP, taking 2.5x total execution time to finish.

The culprint behind such severe performance penality is the necessity of destructive full invalidation to all EPT mappings.
TLB flush instrictions can be catergorized into two kinds, full invalidation (`invept`) and single gVA invalidation (`invvpid/invpcid/invlpg`).
Because hypervisor-based solutions, capturing GPT entries through faulting @vee15heterovisor @isca17heteroos and scanning @eurosys23vtmm, can only see GPT entries and EPT entries, which only contains gPA and hPA, they have to resorts to a full invalidation because the lack of gVA, to ensure the capture of futher PTE.A/D bits.
However, guest-based solution is able to follow the entire process of GPT walk and extract the initial gVA, enabling the use of cheaper single invalidation.
Furthermore, guest-based solutions is now able to leverage more efficient EPT-friendly PEBS instead of TLB flush intensive PTE.A/D bits.

== EPT-friendly PEBS
Contrary to prior assumptions @eurosys23vtmm @osdi24memstrata, we find that PMU sampling is now well-supported with strong isolation for guest states under virtualization @lkml22eptfriendlypebs.
Even cross-architecture support for PMU sampling exists @lkml24riscvguestsampling, allowing extending PMU sampling based hotness classification to a wide arrange of machines in the cloud with simple changes on the collected events.

The main concern for guest PEBS support is the _potential_ breach of isolation boundaries caused by sharing the sample buffer and leaking sensitive information across VMs.
Prior systems intuitively assume that PEBS enabled in the guests will generate samples and write to the host OS's PEBS buffer @eurosys23vtmm, leaking load/store addresses via generated samples. 
However, we found that this is not the case.
The PEBS sample buffer is part of the CPU debug control data structure and is controlled via a debug control register.
Hardware virtualization automatic switches to a guest-private PEBS buffer via `vmcs.debugctl` field in the Virtual Machine Control Structure’s.
Samples generated in executing different VM will be written directly to its private buffer ensuring isolation.

== Scalability
With the avaialbity of PEBS in guests, its intuitive to apply prior PEBS-based designs directly in guests.
However, a machine often have a large number of VMs running concurrently, a direct repurposing would mean compound management overhead.
HeMem @sosp21hemem dedicate one core to pull samples from the PEBS buffer, wasting the total number of VMs worth of CPU cores just on access tracking.

We present a scalability study of eixting both of PTE.A/D (TPP) and PEBS-based (Memtis) designs in @fig-motivation-scalability.
We launch up to nine virtual machines on a 36 core machine and 8.1 billion GUPS transactions with a 126 GiB working set divided evenly across all virtual machines while preserving the access distribution.

Results show that, naively using TPP as guests could waste more than three and a half CPU cores in a 36-core system, while optimized PEBS design like Memtis still waste half a core.
Cloud environments rent CPU resources to customers and often aim for maximal resource utilization.
Such wastage might introduce TCO increase, negating the benefits of memory expansion through a slow memory tier.

```



|        | Total      | `invept`   | Elapsed  | Seconds |
| ------ | ---------- | ---------- | -------- | ------- |
| H-TPP  | 82,504,466 | 20,214,840 | 14:56.35 | 896.35  |
| G-TPP  | 17,707,154 | 0          | 5:53.91  | 353.91  |
| G-Ours | 9,305,363  | 0          | 4:59.57  | 299.57  |

## Design

- Design Concept: Delegating TMM to Guest OS
  1. Can avoid excessive TLB flush
  2. Able to utilize gVA locality
     - gVA locality 只存在于 guest 现有用 PA 空间存在 fragmentation 问题
  3. Able to utilize PEBS
  
  - Implementation Choice: Follow application-transparent (e.g., HeteroOS), but can also support application-tailored (e.g., HeMem)
  
- Efficient and Scalable TMM
  - ~~Ref fig2: PEBS still waste 0.5 cores~~
  - cost per sample?
  - Efficient page migration
  
- Elastic TMP
  - SOTA: hotplugging / ballooning
  - Fine granularity
  - TM awareness
  - Host controlreduce locking and TLB flush (intermediate page)



 



## Implementation

## Evaluation

## Related Works

- PML-based access tracking
  - No address filtering support

- PEBS buf

  ```
  Guest support has existed since early versions of PEBS @intel-sdm.
  However, little attention was paid to it due to an old bug not fixed until version 5 @lkml22eptfriendlypebs.
  Guest memory pages are usually lazily allocated upon #a[EPT] page faults, allowing the hypervisor to overcommit memory beyond the total physical capacity.
  In early PEBS implementations, the sample write process could not be interrupted by #a[EPT] page faults @lkml14silvermontpebs.
  When the PEBS buffer’s memory was lazily allocated or swapped to disk, and a PEBS sample was generated, the system would crash, causing catastrophic failures that could bring down all virtual machines running on the same physical server.
  Although Intel mentioned a workaround @intel-sdm, it required fully populating all memory pages before launching a VM and completely disabling swapping @lkml14silvermontpebs, which was not economically feasible for cloud vendors.
  
  With the introduction of EPT-friendly #a[PEBS] in version 5 @lkml22eptfriendlypebs, #a[PEBS] sample writes can continue after being interrupted by an #a[EPT] page fault.
  Hypervisors no longer need to pre-allocate all memory pages or disable swapping, making #a[PEBS] a free bonus feature to enable in the cloud.
  This advancement allows guest operating systems to leverage instruction sampling efficiently and securely, facilitating low-overhead hotness tracking necessary for effective tiered memory management in virtualized environments.
  ```


## Discussion

- HeMem



---

#### Raw data

|          |                          | Host       | Guest      | Ours      |
| -------- | ------------------------ | ---------- | ---------- | --------- |
| Root     | `tlb_flush `             | 58697567   | 1330       | 1029      |
| Root     | `remote_tlb_flush`       | 20214840   | 0          | 0         |
| Non-root | `nr_tlb_local_flush_one` | 2976774    | 2752613    | 2753576   |
| Non-root | `nr_tlb_remote_flush`    | 602668     | 14929754   | 6508334   |
| Non-root | `nr_tlb_local_flush_all` | 12617      | 23457      | 42424     |
|          | Total flush              | 82,504,466 | 17,707,154 | 9,305,363 |
|          | GUPS Elapsed             | 14:56.35   | 5:53.91    | 4:59.57   |