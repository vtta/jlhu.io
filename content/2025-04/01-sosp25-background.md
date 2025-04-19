+++

+++

## Background

- 2D translation and TLB

  - Necessity: separated address space for each VM
  - Architectural support: 

- TMM

  - Definition

  - hypervisor-based 

    - Tracking
    - Classification
    - Migration 

  - host-based 

    - PTE.A bit based + MMU notifier
    - PEBS

#### Version 2

```

```

 #### Version 1

```
= Background
In the section, we explore the complexities of address translation in virtualization and examine how existing tiered memory management approaches attempt to optimize performance and capacity trade-offs in both virtualized and non-virtualized environments.
== Virtualized Environments
In virtualized environments, each virtual machine operates within an isolated guest physical address space.
However, since all virtual machines share the same underlying hardware, modern processors incorporate an additional layer of address translation to facilitate physical memory virtualization.
This translation is implemented through Extended Page Tables (EPT) in Intel's `x86_64` architecture, mapping guest physical addresses (gPA) to host physical addresses (hPA).
Combined with the guest's own virtual-to-physical address translation through guest page tables (GPT), each application memory access undergoes a two-dimensional (2D) page table walk.

This 2D translation process introduces significant overhead.
Modern Intel processors utilize 52-bit gPAs and 57-bit hPAs with a 4KiB page size.
Because page table entries (PTEs) are 8 bytes each, a single page can contain only 512 entries, necessitating 5-level translations for both dimensions—potentially requiring up to 25 memory accesses for a single address translation.

To mitigate these translation overheads, Translation Lookaside Buffers (TLBs) cache translation results.
Modern TLBs can cache both 1D mappings (gVA to gPA) and flattened 2D mappings (gVA directly to hPA). @intel-sdm
In the optimal scenario, a TLB hit from within a virtual machine directly yields the target hPA, bypassing the expensive page table lookups.
However, when TLB flushes occur, both 1D and 2D mappings are invalidated, requiring up to 25 page table lookups to repopulate the cache—a significant performance penalty.

== Tiered Memory Management Systems
Tiered memory systems present applications with a unified virtual address space while transparently managing data placement across different physical memory media with varying performance characteristics and capacities.
Tiered memory management systems dynamically monitor access patterns and adjust data mapping from application virtual addresses to physical media to optimize performance while leveraging the capacity benefits of all available media.
This approach capitalizes on the observation that application data exhibits varying access frequencies;
placing frequently accessed "hot" data in faster media and rarely accessed "cold" data in slower media can achieve both performance and capacity objectives.

Recently, tiered memory has attracted extensive research interest, with host-based systems leading development @asplos17thermostat @asplos19nimble @sosp21hemem @atc21autotiering @asplos22tmo @asplos23tmts @asplos23tpp @sosp23memtis @osdi24nomad @sosp24colloid @eurosys25pet, while hypervisor-based solutions @vee15heterovisor @socc16raminate @isca17heteroos @eurosys23vtmm @osdi24memstrata have received comparatively less attention.
Existing management processes typically involve three key steps: access tracking, hotness classification, and data migration.

Hypervisor-based solutions predominantly rely on Page Table Entry Access/Dirty (PTE.A/D) bits for access tracking.
During address translation for memory accesses, the A/D bits are set within each level of the page table.
In virtualized systems, bits in both GPT and EPT are affected.
Previous work such as HeteroVisor @vee15heterovisor and HeteroOS @isca17heteroos leverage PTE.A bits from GPT, trapping every modification to record access information.
vTMM @eurosys23vtmm also uses guest-side A bits but passes GPT pages from guest to hypervisor for intrusive scanning.
RAMinate @socc16raminate relies on software scanning of PTE.D bits from EPT.

After aggregating access information across multiple scanning rounds, these systems classify page hotness by sorting access frequencies @eurosys23vtmm or using LRU data structures @vee15heterovisor @isca17heteroos.
The hottest data is then migrated to fast memory (FMEM) either at the hypervisor's discretion @socc16raminate @eurosys23vtmm or exported to guests @isca17heteroos.

Modern host-based solutions support A-bit hotness tracking through Linux derivatives @asplos19nimble @atc21autotiering @asplos23tmts @asplos23tpp @eurosys25pet and have introduced more efficient hardware-based access tracking mechanisms, including Processor Event-Based Sampling (PEBS) @sosp21hemem @sosp23memtis.
While repurposing host-based solutions in the hypervisor host OS might seem intuitive, potentially leveraging Linux's MMU notifier mechanism to obtain EPT.A bits, advanced hardware-based solutions like PEBS cannot be directly applied in virtualized environments because host-side PEBS cannot collect guest samples.

```



#### Draft

```
In this section, we discuss the classic pipeline of tiered memory management. We then discuss virtualized environemnts and how it affects such pipeline.

= Background
== Virtualized Environments
In a virtualized environments each virtual machine is presented with an isolated guest physical address space.
However, because all virtual machines share the same underlying hardware, an additional layer of address translation is incooperated by modern processor to facilitate physical memory virtualization.
Such translation is supported by Extented Page Tables (EPT) in Intel's x86_64 architecture, mapping guest physical address (gPA) to host physical address (hPA).
Combing with guest's own virtual (gVA) to physical (gPA) address translation through guest page talbe (GPT), each application memory access would go through a 2-dimentional (2D) page table walk.

Such 2D translation can become extremely expensive.
Modern Intel processors uses a 52 bit gPA and 57 bit hPA with a page size of 4KiB.
Because pagetable entries (PTE) are of 8bytes, each page can only contains 512 entries, requiring 5-level translation for both the two dimensions. 

To hide such translation overheads, TLBs are incooperated to cache translation results.
Modern TLBs are able to cache both 1D mapping from gVA to gPA and flatten 2D mapping from gVA directly to hPA.
In the best scenario, a TLB hit from within a virtual machine would directly find the target hPA bypassing at most 5x5 page table lookups.
The downside is, when TLB flush happens, both the 1D and 2D mapping would be invalidated and require upto the full 25 page table lookups to repopulate the cache.

Virtualized clouds consolidates large amount of virtual machines on shared hardware and exploit overcommitment for profit.
Cloud tenants rent virtual machines and choose resources spec and service tiers according to their expected maximum usage and workload criticality.
However, not all virtual machines will use up their maximum resources simultaneously, allowing cloud vendors to overcommit available hardware resources to maximize infrastructure utilization and profitability.
To be able to keep up with tenants' resource request when usage increases, cloud should maintain elasticity.
During request spikes, cloud should also be able to maintain QoS control among service tiers enforcing resources assignment to critical services.

== Tiered Memory Management Systems
In a tiered memory system, applications issue load store instructions to a unified virtual address space as usual, but those load stores might finally touch the different underlying physical memory media through different interconnect composed of different technologies with different performance and capacity.
It's the tiered memory management systems' duty to monitor and alter the data mapping from application virtual address space to the final physical media in order to enjoy the performance and capacity benefits from all media.
Such a system leverages the common observations that application data are often have different access freqeuency, by placing frequently accessed hot data to the fastest media and rarely accessed cold data to the slow media would achieve the joint performance and capacity objectives.

Currently, tiered memory has attracted extensive interests by host-based systems @asplos17thermostat @asplos19nimble @sosp21hemem @atc21autotiering @asplos22tmo @asplos23tmts @asplos23tpp @sosp23memtis @osdi24nomad @sosp24colloid @eurosys25pet while hypervisor-based solutions @vee15heterovisor @socc16raminate @isca17heteroos @eurosys23vtmm @osdi24memstrata are lagging behind.
Existing management process commonly involve three steps, i.e. access tracking, hotness classification and data migration.

Hypervisor-based solutions often leverage PTE.A/D access information to track hotness.
Page table walk of the address translation process for a load/store memory access would set set the Access/Dirty bit inside each level of pagetable.
In a virtualized systems, bits from both the GPT and EPT will be set.
HeteroVisor @vee15heterovisor and HeteroOS @isca17heteroos leverages PTE.A bits from GPT.
They trap every PTE modification to GPT and record A bits information.
vTMM @eurosys23vtmm also relies guest-side A bits, but instead of trapping, it passes GPT pages from guest to hypervisor for intrusive scanning.
RAMinate @socc16raminate relies on software scanning of PTE.D bits from EPT.

After aggregating access information from two or more rounds of scanning they perform hotness classification through sorting page access frequency @eurosys23vtmm or use LRU data structure @vee15heterovisor @isca17heteroos.
Hotest data are then migrated to FMEM at the hypervisor's discretion @socc16raminate @eurosys23vtmm or exported to guests @isca17heteroos.

Emerging host-based solutions also supports A bits hotness through linux-deratives @asplos19nimble @atc21autotiering @asplos23tmts @asplos23tpp @eurosys25pet, morely interestingly they have proposed more efficent hardware-based access tracking, including PEBS @sosp21hemem @sosp23memtis.
Direct repurposing host-based solutions in the hypervisor host OS is an intuitive solution.
It is possible to leverage MMU notifier mechanism of Linux kernel to let linux-derative solutions obtain EPT.A bits hotness.
However, more advanced hardware-based solutions are not applicable.
Because PEBS on the host cannot collect guests samples.

```

[[gPA-hPA]](https://hhb584520.github.io/kvm_blog/files/virt_mem/kvm-overview.pdf)





---

## Background<1

### 1. Tiered Memory Management (TMM) Systems

- Overview of tiered memory systems and their importance in modern computing

- Overview of TMM pipeline

  - Access sampling
  - Hotness classification
  - Data migration

- Current approaches and their limitations:

  - Access Sampling:

    - PTE.A bit-based approaches (Nimble, TPP, Nomad)
  - PMU sampling approaches (HeMem, Memtis)
    - Advantages and limitations
    
  - Hotness Classification:

    - Split-LRU approaches (Linux derivatives)
  - Frequency-based classification (AutoTiering, Memtis)
  
- Data Migration:
  
    - Promotion mechanisms (NUMA hinting faults)
- Demotion approaches (watermark-based, memory reclamation)

### 3. Hypervisor-Based Tiered Memory Management Systems

- Provide historical context of hypervisor-based approaches
- Compare the approaches of key systems: HeteroVisor, HeteroOS, RAMinate, vTMM, Memstrata
- Highlight their assumptions about guest OS capabilities
- Present a table comparing these systems

---

## Motivation <1.5

### 1. TLB flush overhead

- Explain how virtualization adds complexity to TMM with an additional layer of address translation
- Discuss the performance implications of hypervisor-based approaches:
  - Overhead of frequent context switches in para-virtualization (HeteroVisor, HeteroOS)
  - Translation overhead in hardware virtualization approaches (vTMM, RAMinate)
- Introduce your key insight: disaggregating data placement from memory provisioning
- Emphasize how this aligns with cloud performance and efficiency requirements

### 2. PMU Sampling Support under Virtualization

- Clarify the misconceptions about PMU sampling in virtualized environments
- Explain Intel's PEBS implementation and how it's actually virtualization-ready
- Detail the architectural components that enable secure, isolated PMU sampling in guests
- Discuss the historical challenges and recent advancements (EPT-friendly PEBS)

## Design Objectives and Challenges

### 2. Tiered Memory Provisioning to VMs (Challenge 2)

- Discuss limitations of static memory provisioning approaches
- Analyze the challenges of existing dynamic approaches:
  - ACPI table exposures and their inflexibility
  - virtio-mem and memory hotplugging granularity issues
- Highlight the need for elastic, fine-grained memory provisioning
- Emphasize how this impacts cloud resource utilization efficiency

### 3. Scalable Guest Management (Challenge 3)

- Present your experimental findings (Fig. 1) showing overhead of existing OS-level TMM in VMs
- Emphasize how management overhead scales with VM count
- Connect this to cloud economics (TCO increase, wasted CPU resources)
- Explain why naively applying existing OS-level TMM solutions isn't viable in dense VM environments

### 4. Design Objectives (Transition to Design Section)

- Summarize the three key challenges identified
- Present two-part solution preview:
  - HyperFlex for elastic memory provisioning
  - HyperPlace for lightweight, scalable guest-based TMM
- Highlight how to addresses cloud requirements for efficiency, elasticity, and scalability

---

## Design





---

## Related Works

## Discussion