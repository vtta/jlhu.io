+++

+++

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

#### Version 1

```
= Design
== Guest-Delegated Tiered Memory
We present _Guest-Delegated Tiered Memory_, a novel tiered memory architecture for virtualized environments that achieves efficiency, elasticity, scalability, and flexibility.
Our key insight is to delegate the entire tiered memory management (TMM) pipeline to guest virtual machines while preserving only tiered memory provisioning (TMP) inside the hypervisor.

With this delegation approach, access information comes directly from within guest virtual machines, eliminating the need to tediously extract TLB flush-intensive PTE.A/D bits hidden deeply inside the memory virtualization process of 2D paging.
Guest virtual machines can now utilize readily available EPT-friendly PEBS hardware directly exposed to guests, providing ready-to-use load/store addresses.
While guest-delegated tiered memory maintains application transparency, it also enables the flexibility for tenants to apply custom-tailored management schemes if desired.

However, to effectively support multi-tenant cloud environments, guest-delegated TMM requires careful design considerations to maintain efficiency and scalability, while hypervisor-supported TMP enables the elasticity demanded by cloud environments.

== Efficient and Scalable TMM
#include "figure/design-address-space.typ"

Efficiency and scalability in our design stem from properly exploiting locality information with minimal tracking overhead.
Contrary to the conventional wisdom of classifying hotness in physical address spaces, we leverage the insight that guest virtual address space retains the most locality information, undisturbed by fragmentation within the kernel's page allocator.
To demonstrate this, we ran a real-world application, XSBench, with a 12 GiB memory footprint inside a guest virtual machine with 4 cores and 16 GiB of usable physical memory.
We used a memory access profiling tool, DAMON @middleware19daptrace @hpdc22daos, to record the access hotness distribution across both virtual and physical address spaces over time, as shown in @fig-design-address-space.
The results reveal that in virtual address space, the hottest region is contained in a small range with strong spatial locality.
However, in physical address space, hot spots appear scattered across the entire usable memory range.
This scattered pattern is introduced by the lazy physical page allocation policy employed by OS kernels.
Physical pages are not assigned to virtual addresses until first access, forcing physical page allocation to follow access sequence rather than spatial locality, thus clobbering the locality seen in virtual address space.

=== Range-based Hotness Classification
#include "figure/design-classification.typ"

To address this issue, we designed a range-based hotness classification scheme that operates in the guest virtual address space to preserve maximum locality information.
This contrasts with the physical-page-centric hotness classification methods employed in existing tiered memory systems, which operate in physical address space.
Our range-based hotness classification utilizes a segment-tree-like structure that divides and identifies virtual addresses into ranges of interest, as illustrated in @fig-design-classification.
Instead of tracking hotness at a fine-grained page granularity for all pages in the entire physical address space, our range-based solution minimizes access sampling and tracking overhead spent on cold memory by grouping cold memory into large regions.
Simultaneously, it maximizes tracking accuracy on hot memory by splitting more frequently accessed ranges into exponentially smaller sub-ranges.
Hotness is then ranked by each range's average access frequency and age to facilitate optimal data placement.

*Range Split.*
We discover the hottest ranges through a series of split operations.
After collecting an epoch's worth of samples, we check whether any ranges need to be split.
We examine all the leaf-level ranges to determine if any have significantly more accesses than both of their neighbors.
Since memory accesses can come from all CPU cores, we define a significance factor
$ alpha = (Delta"access")/(tau_"split" dot "vcpu")$,
where $Delta"access"$ is the difference in memory accesses compared to a neighbor,
$tau_"split"$ is the split threshold and $"vcpu"$ is the total number of virtual CPUs.

When a split occurs, we divide the range in the middle into two halves with equal sizes.
The access count for each new range is set to half the total access count of the original range.
We do not split ranges beyond the split granularity, currently set to 2 MiB.
Each range also has two age fields that track when it was created during the split operations and when its accesses were last found.
During each epoch, the access count for each region is halved to ensure older accesses gradually decay to zero.

*Hotness Ranking.*
After the splitting process, we calculate each range's hotness frequency by dividing its total access count by its size.
We rank all the ranges based on these hotness frequencies.
If frequencies are equal, we then use the range's creation age as a tiebreaker, leveraging temporal locality, as newly created ranges might be accessed more frequently in the near future.
We aim to fit as many hot ranges as possible into FMEM based on current capacity and the access distribution calculated earlier.

Our hotness classification process is agile, capable of processing TiB-scale virtual address spaces in seconds.
With a 500 ms epoch length, reaching the smallest hot spots of 2 MiB size inside a 64 TiB space requires approximately 24 split operations within 12 seconds, creating fewer than 50 ranges, which takes negligible time to manage or rank.
The application's virtual address space is sparse, resulting in only a few deep branches of small ranges in the tree, while the rest are large ranges with infrequent accesses.
The total number of ranges is expected not to exceed several hundred.
Ranges can also be merged to further reduce the total number of ranges to manage.
If neighboring ranges' access counts have decayed to zero and there have been $tau_"merge"$ splits since then, they are merged into a single range.

=== EPT-friendly PEBS
Achieving guest virtual address hotness management requires application access tracking in this exact space, which can be directly supplied by broadly available EPT-friendly PEBS.
Our design eliminates additional address translation costs through efficient sample collection with EPT-friendly PEBS.

PEBS was traditionally believed to be unavailable in guest mode @eurosys23vtmm @osdi24memstrata.
We discovered that the root cause was an architectural bug @lkml14silvermontpebs where the PEBS write process could not be interrupted by EPT page faults without risking machine malfunction.
Cloud environments require memory overcommitment, in which guests' memory is lazily allocated and corresponding EPT entries are lazily populated at the EPT page fault triggered by first access.
Although guest PEBS could technically be enabled by avoiding this architectural defect through eagerly mapping all available memory assigned to a VM and disabling swapping, with the recently introduced PEBS version 5 @lkml22eptfriendlypebs, this bug no longer exists, enabling painless EPT-friendly PEBS.

The PMU operating under guest mode hardware captures load/store samples with guest virtual addresses and writes to the PEBS buffer.
Despite the directly available and favorable virtual address samples, previous PEBS-based designs like HeMem and Memtis opted to use physical addresses and record physical page hotness inside an auxiliary page table structure.
This approach requires walking the page table and translating the virtual address to physical address for every single valid sample generated by PEBS, introducing address translation costs that hurt efficiency and scalability.
Our design feeds the readily available virtual address samples directly to range-based hotness classification without any address translation.

Beyond address translation costs, sample collection costs also constitute a significant portion of overall management overhead @sosp23memtis.
Prior works either dedicated one core for sample collection through busy polling @sosp21hemem or assigned the collection process with a CPU overhead budget @sosp23memtis.
However, our findings in @fig-motivation-scalability show that the tracking overhead per VM often overshoots the 3% CPU budget employed by Memtis due to the feedback delay in Memtis' sample frequency adjusting scheme, frequency is not lowered until it overshoots the perf buffer and overwhelms the collection thread.
To address this, we choose a small and constant sample frequency to avoid overshoots and integrate our sample collection process into process context switches instead of using dedicated threads.
Samples are drained right after switching out from the generating process and fed to range-based classification through a lock-free multi-producer single-consumer channel.

*Event Selection*
Memory tiers can be composed of different media with varying PEBS support capabilities.
Instead of using traditionally employed media-specific cache miss events like `MEM_LOAD_L3_MISS_RETIRED`, which only supports DRAM and PMM, we use the memory load latency event `MEM_TRANS_RETIRED.LOAD_LATENCY` as our PEBS trigger.
This approach allows us to capture access information in a media-agnostic manner, maintaining compatibility with emerging CXL memory.

Using a single memory load latency event enables us to capture access information from both FMEM and SMEM tiers simultaneously, reducing management overhead compared to cache miss events, which would require at least two separate events for a two-tiered system.
One challenge with the load latency event is its inability to distinguish between cache hits and actual memory accesses.
However, we address this limitation through the built-in filtering feature of the load latency event via the `MSR_PEBS_LD_LAT_THRESHOLD` register.
This register allows us to specify a latency threshold such that only memory operations exceeding this threshold generate PEBS samples.
On our evaluation system, Intel's Memory Latency Checker reports typical cache hit and memory read latencies of 53.6ns and 68.7ns, respectively.
By setting our load latency threshold to 64ns, we effectively filter out cache hits while capturing genuine memory access samples, ensuring our hotness classification is based on actual memory tier interactions rather than cache behavior.

=== Symmetric Page Exchange
The final stage in our TMM pipeline is data migration through symmetric page exchange.
This mechanism ensures optimal placement of hot and cold pages across memory tiers while minimizing system overhead.

Following the hotness ranking process (described in @design-classification), we identify the optimal distribution of pages between FMEM and SMEM.
Specifically, we determine the largest index $f$ such that the total number of pages in ranges with indices $[0, f)$ does not exceed the available FMEM capacity. Our exchange process then occurs in three phases:
_❶Promotion candidate identification:_
We traverse the process's page table within hot ranges (indices $[0, f)$) to identify pages incorrectly placed in SMEM.
These misplaced pages are collected into a promotion list of length $m$, representing FMEM-deserving pages currently residing in the slower tier.
_❷Demotion candidate identification:_
We examine coldest ranges (with largest indices) to collect exactly $m$ pages incorrectly placed in FMEM, forming a demotion list with length equal to the promotion list.
This one-to-one correspondence enables our efficient symmetric exchange approach.
_❸Batched symmetric exchange_
between the promotion and demotion lists.
Pages are first unmapped from their respective page tables, then their contents are swapped directly, and finally, they are remapped to their new locations.
This batched approach minimizes the number of page table locks acquired and TLB flushes required.

Our symmetric exchange design offers advantages over previous tiered memory migration approaches through maintaining stable memory usage, and preventing memory reclamation.
Unlike systems that employ sequential migration with temporary buffers @osdi24memstrata @eurosys23vtmm @atc21autotiering, our method eliminates the need for intermediate pages or transient memory allocation during the migration process.
Traditional approaches first demote a page to a temporarily allocated buffer to create space, then promote another page into the vacated slot.
This sequential process can trigger memory pressure that activates the kernel's page reclamation mechanisms, either through background reclamation via `kswapd` or through direct reclamation on the critical path.
These reclamation events increase CPU overhead and can cause unwanted cascading demotions of potentially hot pages.

== Elastic TMP
Our guest-delegated approach requires the hypervisor to focus on efficient and elastic tiered memory provisioning. 
We achieve this through a combination of NUMA node interfaces and a novel double balloon mechanism called Demeter balloon.

*NUMA-Based Tier Exposure.*
To enable tiered memory awareness in guest operating systems without introducing new abstractions or requiring application modifications, we leverage the existing NUMA node interface.
During virtual machine boot, we expose two virtual NUMA nodes corresponding to the FMEM and SMEM tiers on the host machine through the virtualized ACPI table.
The relative performance characteristics between tiers are also conveyed through the ACPI table using NUMA distance values, enabling guests to construct an accurate tier topology.

*Demeter Balloon Mechanism.*
To enable dynamic memory resizing in virtualized cloud environments, we developed the Demeter balloon mechanism.
Unlike conventional memory hot-plugging approaches @lkml05hotplug that operate at coarse block granularity (128MiB for x86-64 or 1GiB for aarch64 @vee21virtiomem), our solution assigns each guest NUMA node a dedicated memory balloon supporting page-granular inflation and deflation.
We configure each NUMA node with a maximum capacity equal to 100% of total system memory, allowing memory composition to smoothly transition between any ratio of FMEM and SMEM, from full FMEM to full SMEM.
This page-level granularity enables precisely tailored memory allocation that adapts to real-time workload demands.
The flexible allocation range effectively supports both overcommitment for lower-tier VMs and overprovisioning for higher-tier VMs, significantly enhancing overall system elasticity.
Implemented as a hypervisor-emulated virtual device, the Demeter balloon requires only a simple driver module in the guest OS, facilitating straightforward deployment across diverse cloud environments.

```



#### Draft

```
= Design
== Guest Delegated Tiered Memory
We present _Guest Delegated Tiered Memory_, a novel tiered memory architecture for virtualized environemt with efficiency, elasticity, scalability, and flexibility.
Guest delegation hand over all stages of the tiered memory management (TMM) pipeline to the guest virtual machines and only preserve tiered memory provisioning (TMP) inside the hypervisor.
With delegation, access information could come directly from within the guest virtual machines, instead of tediously extracting the TLB flush intensive PTE.A/D bits hidden deeply inside the memory virtualization process of 2D paging.
Guest virtual machines could now utilize readily EPT-friendly PEBS hardware directly exposed to guests with ready-to-use load/store addresses.
Geust-delegated tiered memory still retains the application transparency but enables the flexibility of applying custom tailored management scheme if tenants desire.

However, to facilate multi-tenant cloud environment, guest delegated TMM requires carefully design considerations to maintain efficiency, scalability, while hypervisor supporting TMP to enable elasticity.

== Efficient and Scalable TMM
#include "figure/design-address-space.typ"

Efficiency and scalability stems from the proper exploitation of locality information with minimal tracking.
Contrary to prior wisdom of classifiying hotness in the physical address spaces, we leverage the insight that gVA space retians the most locality information undisturbed by fragmentation within kernel's page allocator.

To demonstrate this, we run a realworld applicaiton, XSBench, with a 12GiB memory footprint inside a guest virtual machine with 4 cores and 16GiB of usable physical memory.
We use a memory access profiling tool, DAMON @middleware19daptrace @hpdc22daos, to record the access hotness distribution across both virtual and physical address space overtime as shown in @fig-design-address-space.
Results show that in virtual address space, the hottest region is contained in a small range with strong spacial locality.
However, the physical address space show scattered small hot spots spanned across the entire usable memory range.
Such scattered pattern is introduced by the lazy physical page allocation policy employed by OS kernels.
Physical pages are not assigned to virtual addresses until first access, forcing physical page allocation to follow access sequence instead of spacial locality, thus clobber the locality seen in virtual address space.


=== Range-based Hotness Classification
#include "figure/design-classification.typ"

To tackle this problem, we design a range-based hotness classification scheme operates in the guest virtual address space to preserve the maximal locality information, contraty to the physical-page-centric hotness classification method employed in existing tiered memory systems which operates in physical address space.
Range-based hotness classification utilize a segment-tree like structure that divides and identifies virtual addresses into ranges of interest, as illustrated in @fig-design-classification.
Instead of tracking hotness in a fine-grained page granularity for all pages in the entire physical address space, range-based solution is able to minimizing access samples and tracking overhead spent on cold memory by grouping cold memory into large regions while maximize tracking accuracy on hot memory by splitting the more frequently accessed ranges into expoentially smaller sub-ranges.
Hotness are then ranked by each range's average access frequency and age to facilitate data placement.


*Range Split.*
We discover the hottest ranges through a series of split operations.
After collecting an epoch’s worth of samples, we check whether any ranges need to be split.
We examine all the leaf-level ranges to determine if any have significantly more accesses than both of their neighbors.
Since memory accesses can come from all CPU cores, we define a significance factor
$ alpha = (Delta"access")/(tau_"split" dot "vcpu")$,
where $Delta"access"$ is the difference in memory accesses compared to a neighbor,
$tau_"split"$ is the split threshold and $"vcpu"$ is the total number of virtual CPUs.
When a split happens, we will split the range in the middle into two halves with equal sizes.
We experimentally choose an epoch of 500 ms, $alpha$  of 2, and $tau_"split"$ of 15.
When a split occurs, we divide the range into two equal halves.
The access count for each new range is set to half the total access count of the original range.
We do not split ranges beyond the split granularity, currently set to 2 MiB.
Each range also has two `age` fields that track when it was created during the split operations and when its accesses were last found.
During each epoch, the access count for each region is halved to ensure older accesses gradually decay to zero.

*Hotness Ranking.*
After the splitting process, we calculate each range’s hotness frequency by dividing its total access count by its size.
We rank all the ranges based on these hotness frequencies.
If frequencies are equal, we then use the range’s creation age as a tiebreaker, leveraging temporal locality, that newly created ranges might be accessed more frequently in the near future.
We aim to fit as many hot ranges as possible into FMEM based on current capacity and the access distribution calculated earlier.

Our hotness classification process is agile, which is able to process the TiB scale virtual address space in seconds.
With a 500 ms epoch length, reaching smallest hot spots of a 2 MiB size inside a 64TiB space requires about 24 split operations within 12 seconds, creating fewer than 50 ranges, which takes negligible time to manage or rank.
The application’s virtual address space is sparse, resulting in only a few deep branches of small ranges in the tree, while the rest are large ranges with infrequent accesses.
The total number of ranges is expected not to exceed several hundred.
Ranges can also be merged to further reduce the total number of ranges to manage.
If neighboring ranges’ access counts have decayed to zero and there have been $tau_"merge"$ splits since then, they are merged into a single range.


=== EPT-friendly PEBS
To achieve guest virtual address hotness management requires application access tracking in this exact space which can be directly supplied by broadly avaialble EPT-friendly PEBS.
Our design eliminates any additional address translation costs through efficient sample collection with EPT-friendly PEBS.

PEBS was traditionally believed to be not available in guest mode @eurosys23vtmm @osdi24memstrata.
We find that the root cause was an architectural bug forbiding the PEBS write process to be interrupted by EPT pagefaults @lkml14silvermontpebs or the whole machine could malfunction.
Cloud environments require memory overcommitment, in which guests' memory are lazily allocated and corresponding EPT entries laizy populated at the EPT pagefault triggered by first access.
Although guest PEBS could be enabled by avoding such architectural defect through eagerly mapping all avaialble  memory assigned to a VM and disable swapping, with recent introduced PEBS version 5 @lkml22eptfriendlypebs such bug no long exists, enabling painless EPT-friendly PEBS.

The PMU operating under guest mode hardware captures load store samples with guest virtual address and write to the PEBS buffer.
Despite the directly avaialble and favorable virtual address samples.
Previously PEBS-based designs, HeMem and Memtis opts to use physical address and record physical page hotness inside a auxiliary pagetable structure, which requires walking pagetable and translate the virtual address to physical address for every single valid sample generated by PEBS, introducing address translation costs, hurting efficiency and scalbility.
Our design feeds the readily available virtual address sample directly to range-based hotness classification without any address translation.

Apart from address translation costs, sample collection costs also constitute the great portion of overall management overhead @sosp23memtis.
Prior works either delicate one core for sample collection through busy polling @sosp21hemem or 
assign the collection process with a CPU overhead budget @sosp23memtis.
However, our finding in @fig-motivation-scalability show that the tracking overhead per-VM often overshoot the 3% CPU budget employed by Memtis because of the feedback delay in Memtis' sample frequency adjusting scheme, frequency will not be lowered until it overflows the perf buffer and overwhelm the collection thread.
Thus, we choose a small and constant sample frequency to avoid overshoots and integrate our sample collection process into process context switches instead of using delicated threads, samples are drained right after switching out from the generating process and fed to range-based classification through a lockfree mpsc channel.

*Event Selection.*
Memory tiers could be composed of different media with varing PEBS support.
We use memory load latency event `MEM_TRANS_RETIRED.LOAD_LATENCY` as the PEBS event, instead of traditionally used media-specific cache miss events `MEM_LOAD_L3_MISS_RETIRED` which only supports DRAM and PMM.
Using load latency event allow us to capture access information in a media-agnostic manner maintaining compabilities with future CXL memory.
Using a single memory load latency event is able to capture access information from both FMEM and SMEM tiers, reducing the management overhead compare to cache miss events, which require at least two events for a two tiered system.
The problem with the load latency event is that it can no longer distinguish between a cache hit or a real memory access.
However, such problem is fixable with the built-in filtering feature of load latency event through `MSR_PEBS_LD_LAT_THRESHOLD` register.
Only load latency greater than the specified threshold will be counted and captured by PEBS hardware.
On our system, typical cache hit and memory read latencies are reported to be 53.6ns and 68.7ns by Intel's Memory Latency Checker @intel-mlc.
Thus, we choose a load latency threshold of 64 to filter out cache hits and capture actual memory access samples.


=== Symmetric Page Exchange
We use symmetric page exchange to migrate misplaced data in the last step of the TMM pipeline.

After ranking all managed ranges according to their hotness in @design-classification, we tries to ensure the top `f` ranges are placed in FMEM and the bottom `s` ranges are placed in SMEM.
To maximize the FMEM utilization, we try to find the largest index `f` such that the total number of intersecting pages in ranges with indices in `[0, f)` is not larger than the total fast memory capacity.
We traverse the process’s page table intersecting those ranges to identify pages misplaced in SMEM, collect them into a promotion linked list, the length of the promotion list is the FMEM misplaced page count, numbered as `m`.
We then traverse the bottom `s` ranges to collect `m` pages misplaced in SMEM to form a demotion list in a similar manner.

We perform batched symmetric page exchange on the promotion and demotion list, where candidates are unmapped from their page tables, data are swapped, and finally remapped, minimizing locking on pagetable structures and TLB flushes required.
What's more, unlike paired migration with temporary allocated intermediate pages employed by previous work @osdi24memstrata @eurosys23vtmm @atc21autotiering, where for each page, they first perform demotion to a temporarily allocated free page to make space for future promotion, then premote to the space empited out, our design require no memory allocation, and there is no fluctuation in free memory watermarks that might wake up `kswapd` or trigger on-critical-path direct reclamation to free up fast memory, triggiring no unwanted demotion.

== Elastic TMP
Delegating data placement responsibilities to guest operating systems leaves the hypervisor with the challenge of elastic tiered memory provisioning, we achieve this through NUMA node interface and a double balloon mechanism called Demeter balloon.
To enable the tiered memory awareness of guest delegated management, without introducing new abstractions which needs application modification, we make use of existing NUMA node interface.
During virtual machine boot, we expose two NUMA nodes corresponding to FMEM and SMEM on the host machine through the virtualized ACPI table, relative performance between tiers are also exposed through ACPI table using NUMA distance enabling guest to form the correct tier topology.
To facilitate virtualized cloud's dynamic memory resizing, we design Demeter balloon, associate each guest NUMA node with a memory balloon, enabling page granular inflation and deflation, contrary to virtual block-granular resizing using memory hotplugging @lkml05hotplug.
Such solution provisions memory in blocks, allowing dynamically resizing by online and offline memory only in a block granularity, where, current Linux only supports a 128MiB hotplug granularity for x86-64 architecture or even 1GiB for aarch64 @vee21virtiomem, which is not able to satisfy the elasticity need.
The maximal capacity for each node is set to the maximal total memory capacity, allowing the actual memory composition to range from 100% SMEM to 100% FMEM, supporting both overcommitment for VMs from the lower service tier or overprovisioning for VMs from the higher service tier.
Demeter balloon is a virtual device emulated by the hypervisor.
Guest operating systems need to only load the Demeter balloon driver module to recognize this device.

*Fully Asynchrony.*
Demeter balloon builds upon Virtio @virtio-spec and kernel's `workqueue` executor while the device emulation utile `epoll` to realize a fully asynchronous architecture.
When a hypervisor initialize an inflation or deflation request, it post such request onto the underlying VirtIO queue.
Demeter balloon in guest OS is signalled by a VirtIO interrupt, 
it fetches the request from the queue and identify the corresponding balloon.
Actual memory inflation and deflation is carried out on the `workqueue` asynchronous executor, where Demeter balloon reserves or restores the requested amount of memory pages and post addresses to the corresponding VirtIO queue.
The hypervisor listens to these queues using `epoll()` on their associated event file descriptors `event-fd`.
Upon receiving the page addresses, the hypervisor removes or restores the physical memory backing for these pages.
Finally, the hypervisor signals the completion of each request by posting an acknowledgment on the VirtIO queues.

*QoS Policy Support.*
In addition to controlling the provisioned FMEM and SMEM through inflation and deflation, we additionally expose guest memory statistics to the hypervisor via a statistics VirtIO queue.
These statistics enable machine- or cluster-wide memory schedulers to rebalance FMEM and SMEM across different virtual machines, ensuring quality of service.
We leave the characterization of cloud workloads and design of specific QoS policies to future work.

== Limitations
=== File Page Cache
// Our gVA space management does not cover file page cache which is managed in physical address space.
// However, we believe placing disk data onto SMEM already presents significant performance improvements compared to frequent swapping and triggering IO operations.
=== Intra-hugepage Skewness
// While Demeter does not directly address intra-hugepage skewness by setting split granularity is set to 2 MiB as proposed by Memtis @sosp23memtis.
// Changing split granularity to 4 KiB would highlight such skewness with introduced management overhead.
// However, we believe using hugepage in virtualized environment is more TLB friendly.
// We leave such performance tradeoff to system administrators who would fine tune such parameters according to their specific workload.
```

