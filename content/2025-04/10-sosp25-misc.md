+++

+++



### Related works



#### Version 1

```
= Related Works
== Kernel-based TMM
Kernel-based tiered memory management solutions have dominated recent research, offering various approaches to optimize the performance-capacity tradeoff in heterogeneous memory systems.
These systems have evolved across several dimensions: access tracking mechanisms, migration strategies, management granularity, and optimization objectives.

Early systems like Thermostat @asplos17thermostat pioneered page hotness tracking within the kernel, introducing hugepage awareness for cold page detection, while Nimble @asplos19nimble extended this with support for migrating entire hugepages between tiers.
AutoTiering @atc21autotiering further expanded the design space by supporting four memory tiers rather than the traditional two-tier approach, demonstrating the scalability challenges in multi-tier environments.
Several systems have explored architectural improvements and novel management strategies.
TMO @asplos22tmo investigated the use of swapped memory as a slower tier, while TMTS @asplos23tmts advanced the architectural design through a user-kernel collaborative approach that enables cloud schedulers to dynamically manage tiered memory resources.
TPP @asplos23tpp, which represents the tiered memory support in the modern Linux kernel, introduced proactive demotion strategies.
Recent advances have focused on optimization objectives beyond simple hit rates.
Nomad @osdi24nomad addresses memory thrashing during page migration by allowing shadow copies to exist simultaneously in both FMEM and SMEM tiers.
Colloid @sosp24colloid proposes optimizing for access latency rather than simply packing the hottest data into the fastest memory tier.
PET @eurosys25pet pushes further with aggressive proactive demotion at larger granularity while maintaining low overhead.

Despite these innovations, the majority of kernel-based designs remain built around Linux's page reclamation system and rely primarily on PTE.A/D bits as their hotness information source.
Only Memtis @sosp23memtis has significantly challenged this paradigm by exploring alternative hardware-based hotness sources similar to our approach.
However, unlike our work, these kernel-based systems were not designed for virtualized environments and face fundamental challenges when directly applied to cloud infrastructure, including the TLB flush overhead and EPT compatibility issues we address through our guest-delegated architecture.

== Hardware-based TMM
Hardware-based TMM approaches @fast20empirical @sosp21hemem @sosp23memtis @osdi24memstrata utilize specialized hardware mechanisms for access tracking or offload memory management entirely to hardware.
HeMem @sosp21hemem pioneered PEBS as a high-fidelity hotness source with lower overhead than PTE.A/D bit scanning, while Memtis @sosp23memtis further optimized PEBS-based approaches through dynamic sampling rate adjustment.
In contrast, fully hardware-managed solutions include Intel's PMEM Memory Mode (2LM) @fast20empirical, which uses DRAM as a direct-mapped cache for PMEM without exposing tiers to applications, and Intel's Flat Memory Mode @osdi24memstrata, which exposes combined FMEM and SMEM capacity while managing data placement through hardware.
Unlike these approaches, we demonstrate that EPT-friendly PEBS are now available and can be effectively utilized within guest VMs while providing the elasticity and efficiency needed in virtualized cloud environments.

// Hardware-based TMM approaches @fast20empirical @sosp21hemem @sosp23memtis @osdi24memstrata leverage specialized hardware mechanisms for access tracking or offload entire memory management functions to hardware components.
// These solutions can be categorized into two main groups: systems that utilize hardware performance monitoring units for software-managed tiering, and fully hardware-managed tiering solutions.

// In the first category, HeMem @sosp21hemem pioneered the use of PEBS as a high-fidelity hotness source, demonstrating its ability to capture detailed memory access patterns with lower overhead than traditional PTE.A/D bit scanning. 
// Building on this foundation, Memtis @sosp23memtis further optimized the PEBS-based approach by reducing sample collection overhead through intelligent sampling rate adjustment.
// Both systems showed that hardware sampling can provide superior hotness information while imposing lower CPU overhead compared to traditional software-based tracking methods.

// The second category consists of fully hardware-managed solutions.
// Intel's PMEM Memory Mode (also known as 2LM) @fast20empirical implements a transparent approach where DRAM serves as a direct-mapped cache for slower memory (PMEM), without exposing distinct memory tiers to applications.
// This simplifies deployment but sacrifices flexibility in tier management.
// More recently, Intel's Flat Memory Mode @osdi24memstrata has evolved the hardware-managed approach from caching to tiering, exposing the combined capacity of both FMEM and SMEM to applications while automatically managing data placement through hardware mechanisms.

// Our work differs from these hardware-based approaches in several key aspects.
// Unlike HeMem and Memtis, which were designed for non-virtualized environments, we demonstrate that EPT-friendly PEBS are now available and can be effectively utilized within guest VMs.
// Furthermore, while fully hardware-managed solutions offer simplicity, they lack the elasticity and fine-grained control needed in multi-tenant cloud environments where resource allocation must adapt to changing workloads and service tier requirements.


== DAMON
DAMON framework @middleware19daptrace @hpdc22daos, a memory access profiling tool that has been merged into the Linux kernel, provides monitoring capabilities in both virtual and physical address spaces through a region-based architecture.
While DAMON itself serves as a profiling framework, a separate DAMON-based TMM system @lkml24damontmm has been proposed but remains under development at the time of our work (based on kernel version v6.10).
// A critical limitation of this developing DAMON-based TMM is its fundamental lack of autonomy, it requires explicit user intervention to initiate promotion or demotion actions rather than automatically adapting to changing workload patterns.
// This design positions it primarily as a complementary tool for user-integrated TMM solutions rather than as a comprehensive autonomous system for cloud environments.

The base DAMON framework estimates regional hotness by sampling randomly selected pages' PTE.A bits, with region count determined by static user configuration rather than dynamic workload characteristics.
While useful for manual performance tuning, this approach cannot leverage the high-fidelity sampling data available through PEBS nor adapt automatically to diverse application memory patterns.
Our analysis of the developing DAMON-based TMM implementation reveals additional virtualization-specific limitations:
it relies on PTE.A bits during access tracking, necessitating frequent TLB flushes that introduce substantial overhead in virtualized environments;
it performs hotness classification in physical address space, missing critical locality information preserved in virtual address space;
and it cannot utilize the rich access information available through EPT-friendly PEBS within guest VMs.
In contrast, Demeter provides fully autonomous operation with minimal overhead through its guest-delegated architecture and efficient PEBS-based access tracking.
```



#### Draft

```

== Kernel-based TMM
Kernel-based TMM @asplos17thermostat @asplos19nimble @atc21autotiering @asplos22tmo @asplos23tmts @asplos23tpp @sosp23memtis @osdi24nomad @sosp24colloid @eurosys25pet leads existing TMM development.
With Thermostat @asplos17thermostat introduce hugepage awareness for cold page detection.
Nimble @asplos19nimble further adds page migration support for hugepages.
AutoTiering @atc21autotiering expand mainstream two tiered systems into a four-tierd environment.
TMO @asplos22tmo investiage using swapped memory as the slower tier.
TMTS @asplos23tmts enables cloud scheduler to manage tiered memory through a user-and-kernel-space collaborative design.
TPP @asplos23tpp proposes proactive demotion and represent the tiered memory support in modern Linux kernel .
Nomad @osdi24nomad optimize for memory threshing during page migration though allowing shadow copies to exist in both FMEM and SMEM tiers.
Colloid @sosp24colloid calls for a new management objective of optimizing for access latency instead of packing hottest data on fastest memory.
PET @eurosys25pet urge for even more aggressive proactive demotion with low overhead in a larger granularity.

Despite significant progress made in recent years, kernel-based design revolves around Linux kernel's page reclamation system, with PTE.A/D bits as the major hotness source.
Only challenged by recent design Memtis @sosp23memtis to consider alternative hardware-based hotness sources.

== Hardware-based TMM
Hardware-based @fast20empirical @sosp21hemem @sosp23memtis @osdi24memstrata TMM involve using different hardware access sources or offloading entire TMM to the hardware.
HeMem @sosp21hemem first experiment with PEBS as the hotness source.
While Memtis @sosp23memtis optimize further to reducing sample collection overhead.
PMEM Memory Mode @fast20empirical or called 2LM do not expose memory tiers to application, rather, use DRAM as the direct-mapped cache for SMEM.
Intel Flat Memory Mode @osdi24memstrata, another hardware-based solution switch from caching to tiering, exposing the combined FMEM and SMEM capacity to user while managing data placement automatically through hardware.


== DAMON
DAMON @middleware19daptrace @hpdc22daos is a memory access profiling tool which supports access tracking in both virtual and physical address space.
Despite it utilize a similar region-based architectural, it estimates regional hotness based on a randomly selected page's A-bit information, with user-configured region splits producing only a user-specified number of ranges.
This design may aid manual performance tuning but cannot make use of rich hotness info generated by PEBS, nor could it serve as a comprehensive automatic hotness identification solution for diverse applications in virtualized environments

Recently, a DAMON-based Tiered Memory Management is under development @lkml24damontmm.
At the time of this work, which is based on the kernel version v6.10, we have not seen DAMON-based TMM been merged.
Our preliminary investigation into ongoing DAMON-based TMM development finds out that they utilize PTE.A bits during access tracking and perform hotness classification on the physical address space which exhibits expensive TLB flushes unacceptable in virtualized environment and also could not make use of access information in guest virtual address space provided by EPT-friendly PEBS.
Furthermore, they rely on user-inputted action to perform promotion or demotion with no autonomy and can only serve as a complement to user-integrated TMM design.
```

