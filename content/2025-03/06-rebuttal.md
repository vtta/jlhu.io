+++
+++

总体来看攻击novelty以及evaluation的最多. 没有什么说我们的问题不重要设计不够好.

回应限制2000字.

1. A看起来主要是因为我们没有比较DAMON的range方法觉得novelty不够.
   1. (security)
   2. (performance) I do not find it entirely clear why guest-managed tiered memory would inherently be more performant and scalable.
   3. (cloud)  cloud providers and VM guests sacrificing CPU cycles for memory savings
   4. (motivation) why the guest should be able to make decisions based on observations from the guest’s virtual address space
   5. (novelty) What are the fundamental differences between DAMON?
   6. (evaluation) No hypervisor-based tiering.
   7. (evaluation) GUPS variation, no skewed workload
   8. (evaluation) TPP is best in Graph500
2. B看起来比较nice. 主要也是找的evaluation的问题.
   1. (figure) results hard to parse; figures unreadable
   2. (evaluation) performance is too good to be true
   3. (evaluation) why SOTA so bad?
   4. (evaluation) Are we seeing a pathologically bad workload?
   5. (figure) 4/9/10 should be bigger

3. C看起来是利益相关方. 认为我们guest/host指责分配是HeteroOS的功劳. 得详细比较一下.
4. D看起来是security background. 我们需要用他的思路通过threat model讲一下. 另外D看起来是个好人, 不仅肯定了我们的novelty, 也提出了改进意见.
   1. (security) The threat model is not clear enough.
   2. (security) Security concerns with malicious VMs
   3. (security) Is the hypervisor trusted in your threat model?
   4. (security) How to handle potential manipulations of memory access information by malicious guest VMs?

5. E看起来不太懂, 只关注evaluation以及细枝末节的东西.

感谢reviewer对我们以下贡献的肯定:

- Disaggregating data placement responsibilities from tiered memory provisioning

balbalbalba

我们将address以下问题:

- Security/Threat model: 这个得好好想一下
- DAMON引用的问题, 纯属是忘了. 但是得讲一下我们的range和他的区别.
- 占用guest CPU的问题: 目前大量程序是memory bounded (加引用). 相比超高开销的swapping, 牺牲少量CPU来换取大量可用内存是个不错tradeoff.
- evaluation咋办?
- 图看不清只能帅锅版面不够.
- 单位的问题: fig 4-6 giga-update per second就是单位; fig 6横轴是秒; fig 7单位为秒; fig 8, period的单位就是个, 也算是没单位量? 不够得讲清楚是事件出现次数, 另外横轴是ns和次数; fig 9-10标了秒, 横轴是vm数.

- [ ] 补一下gups zipf
- [ ] Graph500 为啥不好

#### Guest modifications

The guest part of our design is fully contained in the kernel modules. Cloud vendors aleady provide customized linux distributions such as Amazon Linux with custom kernel tunning, we believe bundle additional modules into such distribution requires no guest modifications.

#### GUPS

Our GUPS implementation is a work-stealing variant and is entirely moticated by reproducibility. The original GUPS assigns each thread with a fixed amout of read-modify-write operations at startup, early terminating thread will stay idle and cobbor the execution time and overall throughput measurements.

1. 2.

---

## [Review #522A](https://osdi25.usenix.hotcrp.com/review/522A) 2 | 3

HyperTier is a tiered memory management scheme designed with virtualization in mind. It delegates the memory management operations to the guest, recognizing that guests can safely utilize hardware performance counters. By incorporating range-based hotness detection, leveraging PEBS counters, and implementing a more efficient page migration mechanism within the guest, HyperTier enables effective hot-cold page placement decisions.

### Strengths and weaknesses

Strengths:

- Addresses a timely and relevant problem
- Provides a thorough background and problem-space exploration

Weaknesses:

- Limited ***novelty***. The contributions appear ***incremental***
- The ***motivation*** behind the work could be clarified further
- ***Evaluation***

### Detailed Comments for authors

However, I believe the ==contributions of this work are somewhat incremental==. The paper seems to ==lack some key related work that it builds upon or reimplements==, and the ==evaluation does not provide sufficient evidence to strongly support the proposed benefits==.

#### Motivation

Motivation for shifting the tiering responsibilities to the guest is not entirely clear in the paper:

- Security: Section 3.1 raises security and isolation concerns, specifically regarding risks introduced by the hypervisor. PEBS is presented as a safe option, with the claim that samples are written to GVA, thus, "preventing the hypervisor from accessing the actual location and content of the sample buffer". ==What actually prevents the hypervisor from using the guest's gCR3 value and performing software page walks with EPT to access the guest's information?== A more in-depth discussion and clarification of the isolation and security advantages is needed.

- Performance: The introduction suggests that guest-managed tiered memory can reduce compound overheads introduced by hypervisor-managed tiering. However, I do not find it entirely clear ==why guest-managed tiered memory would inherently be more performant and scalable.== A more detailed explanation would help clarify this point.

  > Hypervisor managed access tracking

- Cloud providers and VM guest model: I found the ==discussion regarding cloud providers and VM guests sacrificing CPU cycles for memory savings to be a little thin.== What motivates cloud providers to expose such trade-offs to the guest? what motivates the guest to sacrifice its own cycles? can't a hyperscaler hide tiered memory behind an SLA agreement, while still being performant?

  > #### Guest CPU
  >
  > Existing cloud vendors like aws and azure already provide memory-optimized varient of virtual machines with cpu to memory ratio cut in half. We believe trading off a small pcentage of CPU cycles in exchange for serveral times more memory capacity is a wellcomed choice.
  
  > #### Host-based design
  >
  > Previous works including HeteroVisor/HeteroOS/vTMM has already demonstrated both a pure host-based and co-operative method bare unacceptable performance issue, failing at the first step of hotness management--access tracking, as introduced in section 3.1. HeteroVisor/HeteroOS trigger pagefaults on every guest pagetable modification; although vTMM also use a hardware-assited method, its PML still trigger VMexit every 512 pagetable modifications. If using PEBS in the host, no guest accesses would be recorded, as PEBS buffer is switched to the guest one during VMentry.
- Missing motivation: There seems to be a gap in the motivation for ==why the guest should be able to make decisions based on observations from the guest’s virtual address space== (of which the paper exploits through gVA region hotness detection).

  > #### Physical address space
  >
  > Application exhibits temporal locality, memory allocated together would often have similar access frequency. However,  fragmentation in Linux's allocator and map-after-touch policy will cause such continuous ranges be maped to scatter physical memory, eliminating locality in physical space.

#### Design & Novelty

- HyperTier appears to be a hybrid of two primary approaches: PEBS-based event counting (similar to Memtis) and range-based hotness detection via virtual address range tracking. The design of HyperTier's range-based hotness tracking, including techniques like range splitting, is strikingly similar to the DAMON range tracking already upstreamed into the kernel. ==Why is DAMON not cited?== What are the fundamental differences between these two approaches, aside from the fact that one uses access-bit scanning and the other relies on PEBS-based mechanisms? A clearer explanation of the novelty in this aspect would be helpful.

  > #### DAMON
  >
  > DAMON \[Middleware'19\]\[HPDC'22\] is a region based memory profiling tools that are both inaccurate and requires user interaction. DAMON relies on a randomly selected page's A-bit access information to estimate the whole region's hotness from which that page is selected. Although DAMON also performs region split, such action is user configured, DAMON will only produce a user instructed number of ranges. Such design might be helpful for performance tunning, but is unable to serve as a full fledged automatic hotness identification and facilate data placement of a wide range of applications.
  >
  >
  > - \[Middleware'19\]Profiling Dynamic Data Access Patterns with Controlled Overhead and Quality
  > - \[HPDC'22\]DAOS: Data Access-aware Operating System

#### Evaluation

- One notable omission is the ==lack of evaluation of hypervisor-based tiering.== The paper could benefit from experiments comparing guest-based tiering with hypervisor-based tiering. For example, by spawning multiple VMs on a hypervisor running a system like TPP - which could leverage the MMU notifiers to trigger KVM to scan the respective EPT entry of the VM's process. Since guest-based tiering is one of the main contributions of the paper, I believe this experiment is crucial for fully evaluating the claims made in the paper.

  > ~~Utilizing MMU notifiers to capture A-bit changes is similar to what HeteroVisor/HeteroOS does, the drawbacks are frequent traps into host kernel and cause sever performance issues. MMU notifiers were traditionally used to maintain shadow pagetable used in software virtualization, reenabling it only to track hotness information would mean throwing away the core benefits hardware virtualization brings, which seems  to be trading the essential for the trivial. **Despite this, we are happy to try this out as a baseline to hightlight our contribution.**  Although we have tried to compare with existing hypervisor-based solutions, they do not open source nor do they provide design details in the paper.~~
  >
  > 接受这个建议, 顺着说. 说为什么能justify我的贡献, 夹带一下trap的问题.
- The micro-benchmark section, which uses a GUPS variation, feels weak. Given that the benchmark uses "hot" and "cold" *contiguous virtual address regions*, it is almost guaranteed that HyperTier will dominate all aspects of performance, as other systems perform these operations at a finer (page) granularity. ==It would be beneficial to have a skewed workload that does not neatly lay out the hot and cold data in different virtual address space regions.== Additionally, this would help explain the disparity between the results from the micro-benchmark and those from real-world benchmarks. More microbenchmarks would provide better insights into the system's behavior across a range of conditions.

  > Our modified GUPS supports zipf distribution, we are happy to include such results. ==Because previous works present GUPS hotset variant ....==
- In section 6.5, the paper claims that HyperTier excels in handling skewed access patterns. Skewed workloads do not always target specific memory regions, but could be specific pages scattered in the virtual address space. For example, in results from benchmarks like ==Graph500==, hot memory accesses are often distributed across the virtual address space, rather than being confined to contiguous regions. This could be a better way to describe why ==TPP outperforms HyperTier in this workload.==

  > #### Skewed workload
  >
  > Other than graph500 we also evaluate other workloads with a skewed access pattern, such as xsbench and bwaves in which we either excel or on par with TPP.
  >

---

## [Review #522B](https://osdi25.usenix.hotcrp.com/review/522B) 3 | 3

HyperTier is a system for dynamic data placement in a multi-tiered memory system, targeted at virtual machine hosting, and dividing the task between an in-VM (guest) component and a hypervisor (host) component. The guest component is responsible for tracking the access frequency of pages and migrating them between fast (local) memory and slow (remote) memory, exposed to it as two NUMA nodes. The host component is responsible for balancing the allocation of fast and slow memory between VMs, using a memory ballooning driver within the guest OS.

Hotness tracking in the guest component is implemented by sampling hardware performance counter events (with PEBS) to estimate the access frequency of memory segments. Segments are dynamically split and merged to minimise bookkeeping overhead on the assumption that hot regions are relatively sparse and clustered. ==The host component supports overprovisioning, but no details are provided on the algorithms used at this level.==

Evaluation is performed on VMs with 16G of guest memory on a system with 128GB local DRAM as fast memory, and two types of slow memory: 512GB of locally-attached Optane DIMMs, and 128GB of NUMA-remote DRAM. Benchmarks seem to suggest a factor of 2 or better throughput improvement on memory-intensive workloads compared to to the next-best alternative included in the comparison.

### Summary of strengths and weaknesses

## Pro

- Attacks a relevant problem with practical implications.
- Solid and well-motivated design choices.
- Thorough background and positioning wrt. related work.

## Con

- ==Presentation of experimental results is very hard to parse== - figures are almost unreadable!
- ==Performance improvement seems too good to be true== (although this *might* be due to the immaturity of the relevant related work).

  > 因为我们主要study kenrel TM能否benefit virtualized TM. 现有工作没法复线. 我们会adopt reviewer A的方案进一步justify .

### Questions for authors’ response

- Why are existing systems so much slower? ==Are we seeing a pathologically bad workload?==

  > 我们都按的相关工作的workload, 没有cherry pick.

### Detailed Comments for authors

The figures are compressed beyond the point of legibility - they need to be significantly bigger. Figures 4, 9, and 10 are particularly bad, and are essentially unreadable.

---

## [Review #522C](https://osdi25.usenix.hotcrp.com/review/522C) 1 | 4 (Sudarsun Kannan, Ada Gavrilovska)?

The paper describes a solution for managing tiered memory in virtualized clouds. The authors claim as contributions principles regarding memory management in virtualized tier memory systems, system components at the hypervisor and guest OS level based on those principles, and their the evaluation and open sourced implementation.

### Summary of strengths and weaknesses

Strengths:

- timely topic,
- good and useful system implementation
- nice contribution to hotness tracking

Weaknesses:

- ==Limited novelty==
- ==Limited evaluation==

### Detailed Comments for authors

I appreciate the work presented in this paper. Open-sourcing access to the functionality presented in the paper is timely given interest in using disaggregated memory in virtualized datacenters.

Particularly valuable are the implementation of the efficient hotness tracking mechanism and guest-hypervisor coordination, which are based on effective use of hardware features for more efficient implementations on hotness tracking and guest-hypervisor interactions.

The use of range filtering to focus hotness tracking to most likely/most relevant memory regions (as opposed to relying on host hints) is an important mechanism.

However, the authors’ ==claim to other significant novel contributions to virtualizing tiered memory systems, are overstated.== The principle of ==separating the **data placement policy** from the low level provisioning of different memory tiers is the ***main approach*** advertised by the ***HeteroOS*** work cited in this paper.== The main difference in that work is that, due to lack of present-day hardware features the design relied on hypervisor support for tracking hotness. The HeteroOS paper discusses leaving placement decisions to the guest, adjusting the local/fast and remote/slow memory capacities across tenants, and other concepts mentioned in this paper. So, while it is still very valuable to adapt these ideas to the features of present-day technologies, the novelty in this paper is overclaimed.

> 说明我们用词问题. 我们实际上想说... 另外我们bridge gap between HeteroOS‘ host hotness info with guest data placement. 我们更往前一步.

> #### Hardware
>
> Hardware support for hotness tracking can be found early as 2008 when Intel introduced PEBS into Nehalem microarchitecture. And PML was also integrated into Linux in 2015. Where HeteroOS is published 2017, we believe
>
> PEBS: Nehalem
>
> <https://lore.kernel.org/all/20150205145248.GA14367@potion.redhat.com/T/>

The evaluation of the work shows the effectiveness of the PEBS-supported hotness tracking combined with the range-based filtering. It seems that ==the end-to-end benefits shown in the paper are really a function of having more or better hotness information, and/or more lightweight way to track hotness.== ==It’s not clear that there are better placement decisions, or better/more elastic sizing decision.== Efficient and effective hotness tracking is an important problem, and my suggestion would be to focus the revision of the paper on this aspect of your contributions.

> 我们就是认为在guest中能做更好的tracking来enable更好的placement. tracking本身就是一个non trivial问题. 我们就是希望kernel-based TM来benefit 虚拟化环境.
> 我们就是在任意一个given sizing情况下做到最好的placement decisions.

------

## [Review #522D](https://osdi25.usenix.hotcrp.com/review/522D) 2 | 3

This paper presents HyperTier, a novel system for managing tiered memory in virtualized cloud environments. HyperTier addresses the limitations of existing hypervisor-based tiered memory management systems by enabling guest operating systems to manage their own memory tiers, while the hypervisor focuses on providing isolated and elastic tiered memory provisioning. The key components of HyperTier include HyperFlex, which provides elastic tiered memory provisioning to guest VMs, and HyperPlace, which optimizes tiered memory management within the guest OS. Evaluation results show that HyperTier can improve performance and efficiency compared to existing systems.

**Strengths**

- Addresses a real problem with an interesting solution.
- Offers elastic tiered memory provisioning.
- Demonstrates significant performance improvements.

**Weaknesses**

- The ==threat model is not clear enough==.
- ==Still some security concerns with malicious VMs.==

### Questions for authors’ response

1. Is the hypervisor trusted in your threat model?
2. How to handle potential manipulations of memory access information by malicious guest VMs?

### Detailed Comments for authors

This paper presents a novel and promising approach to managing tiered memory in virtualized cloud environments. The proposed HyperTier system addresses several limitations of existing systems, such as the lack of isolation and elasticity, and demonstrates significant performance improvements. However, there are a few concerns and suggestions that the authors may want to consider.

==The motivation for isolation in this paper is unclear==, as the threat model is ambiguous. It is mentioned that "memory access patterns, remains confidential, not just from other tenants but also from the hypervisor itself". Does it mean the hypervisor is untrusted? Actually, this ==design itself cannot protect the VM from attacks by a malicious hypervisor==, while previous works have more practical advantages, such as being more transparent and requiring no or fewer modifications to the guest OS. Without Confidential VM or TEE technologies, a malicious host OS/hypervisor can easily access VM data. The paper should clarify the threat model and design goals in Chapter 3, perhaps ==shifting the focus of isolation to "safety and reliability" instead of "security" or "confidentiality."==

==There are some security concerns with malicious VMs.== While this paper claims that HyperTier enhances security and isolation by delegating data placement to guest VMs, it does not fully address the issue of malicious guest VMs which can manipulate memory access information. ==A malicious VM could potentially modify the HyperPlace component or directly change page tables to mislead the hypervisor and gain unfair access to resources.== This could lead to performance degradation or even denial-of-service attacks for other VMs on the same host.

> 我们希望malicious VM不会在placement中gain more advantages. 同样我们的架构天然适合implement defense techniques 比如...

Specifically, ==a malicious guest could deliberately block the HyperFlex virtio front-ends to prevent the hypervisor from reclaiming fast memory (FMEM) through the sub-balloon inflation mechanism.== This could lead to an unfair allocation of FMEM, depriving other VMs on the same host of crucial resources and potentially impacting their performance. [^guest]

Some other questions:

- Challenge-1 in this paper, "Disaggregating data placement responsibilities from tiered memory provisioning", seems more like the key idea of this paper, instead of a challenge.

  > 这个建议很好
- ==It is surprising to see Nomad performs poorly in the test in Figure 4. What are reasons?==

  > 看Figure 7讲清楚, 另外讲他的focus和我们不同. 再细看看.
- There are issues with ==unclear characters or lines in Figures 4, 5, 7, 9, and 10.==
- It seems that the ==GUPS in this paper has been modified? Will such change bring side-efforts?== [^gups]
- In the first paragraph of Section 1.1, it is mentioned that "maximum possible memory per CPU thread has dropped to about one - third of its original capacity", is there some citation? [^drop]
- In Section 4.1, "We optimize the inflation and deflation processes by making them asynchronous reducing total CPU cycles spent on resizing". As far as I know, VirtIO (including virtio-balloon) is also an asynchronous design. [^virtio]

[^guest]: hypervisor有更高级的控制权, 如果virtio被block可以直接让VM停机. 可以在hypervisor中implement一些defense.
[^gups]: 我们是job-stealing gups, 增加了任务分配的公平性.
[^drop]: [23, 24]中十年内从1.5TiB/30thread到4TiB/144thread.
[^virtio]: 我们的设计build atop virtio来实现异步, 额外的异步支持还有...

------

## [Review #522E](https://osdi25.usenix.hotcrp.com/review/522E) 2 | 3

This paper proposes a framework that uses ballooning techniques and a novel replacement/migration algorithm to handle tiered memory in virtualized environments.

**Strengths**

- The paper aims to improve the performance of tiered memory systems in a virtualized environment, which is increasingly important.
- System design seems well-thought-out.

**Weaknesses**

- The ==evaluation section lacks a detailed analysis and discussion of the ***low-level aspect*** of the system.==
- ==Evaluation also lacks a detailed analysis of the ***configuration space*** and the impact on application latency.==
- Many ==graphs== have no units, and some even miss axis legends. [^unit]

### Detailed Comments for authors

Thank you for submitting this paper to OSDI. I enjoyed the key idea of using the performance counters and the replacement/migration algorithms proposed. In general, the paper solves an important solution and proposes a reasonable system design to tackle the problem. However, there are quite a few problems with the evaluation section.

Somehow, most of the graphs in this paper are missing the units, and some graphs do not even have axis legends. Furthermore, in several cases, the captions also do not report the units. This makes all the results hard to interpret without a lot of guessing, which is very problematic for any paper. Furthermore, I found the evaluation section hard to follow for other reasons too. For example, there is a lot of emphasis on presenting end-to-end performance results but very little on analyzing and discussing the low-level metrics that justify the good results. ==There is also a lack of analysis on the configuration space and the impact on application latency (including tail latency) for interactive applications.==

> We believe large memory capacity is mainly needed by analytical applications, for which we mainly presents throughput results. Despite this, we have included a responsiveness evaluation in figure 5, where our design is able to most agilly place hot data shown as the slope of curve.  Our sampling design naturally avoid the batching tradeoff. We actively drain the sample buffer during every context switch, the frequency of which is so high that application should never be delayed due to a buffer full interrupt. What's more, our evaluated applications contains an interactive/transactional database, silo, we are happy to include the average and tail latency results of such application.
>
> The key to our great end-to-end improvement is the low cost hotness tracking and range-based classification, for which we present an overhead breakdown study in figure 7 and a configuration space exploration in figure 8. Our design is able to classify the most amount of hot data in the most agile manner, shown as the peak value and slope in figure 5, while inhibits the lowest overhead, both overall and in the sampling stage, as shown in figure 7. In figure 8, we evaluate the configuration space of both the tracking and classification stages. For the sampling stage, we present a performance matrix across what samples to capture (load latency threshold) and how fast should we sample (sample peroid). And for the classification stage, we presents a similar matrix across how sensitive to hotness difference across ranges (split threshold) and how frequent should we react to collected samples (split period).
>
> In the meantime, system administrators are able to select the most suitable parameters according to such performance matrix across their representative applicaitons. Our tracking parameters depends on PMU hardware. Cloud vendors usually have a large amount of physical machines with similar CPU microarchitectures which share similar PMU hardware. Parameters selected on one platform should be easily expanded to other machines.
>
>

The analysis of application latency is particularly important for such a system because many improvements can be achieved by batching actions (e.g., reducing sampling rates) at the expense of sacrificing application responsiveness (or accuracy), which can be as important or more than throughput.

The **range-based algorithm** proposed seems mostly orthogonal to the use of performance counters and seems generic enough to be usable for general replacement policy in other contexts. Is this the case, or ==is there a reason why this algorithm only works well in the context of virtualized tiered memory?==

> 是为了降低开销设计的算法. 没有imply不能推广.

==Some claims/arguments about the security of the existing and the proposed system are debatable,== to say the least. In particular, the claim that designs that require introspection of the VM to read the guest page tables are insecure because they require the hypervisor to handle sensitive data is problematic. ==In general, security claims or concerns should be made under a concrete threat model, which this paper does not provide.== But there is no objective reason to say that there is a difference in the security of introspection approaches and the proposed approach, given that they both *allow* the hypervisor to read the guest's memory. Whether or not the hypervisor is *required* to analyze it does not affect security. Furthermore, it is typical for the hypervisor to have to read the guest data when migrating pages.

==Who clears the memory when it gets returned from the VM?== [^clean] Is it the responsibility of the guest, the host, or both? It seems hard to imagine that in typical cloud settings either the guest or the host would likely fully trust the other party to do it. So, I would imagine that in a realistic setting, both the guest and host would clear a returned page, which is inefficient.

==There are many configuration options in this paper, and in the end, it is not very clear how dependent these options are on the specific hardware configuration== and, if so, how would an administrator configure the system. This applies to the sampling frequency (for which there is no unit in 4.2) and the "hotness management" algorithms.

> fig 8中explore了最重要的几个param

The concern with the cycles reported in 4s.2 is not clear. Why is there a need to consume the entire CPU budget? If a certain low sampling frequency is enough to produce accurate results, would it not be a plus that the system consumes less CPU resources than the budget allows?

> 每个vm都做自己的tracking, 希望整机开销最低.

[^unit]: GUPS 本身就是unit
[^clean]: Balloon driver takes care of the cleaning.

---

## Junk

> A core insight bethind our work is that existing hypervisor-based hotness tracking all comes with great performance penalties.  HeteroVisor/HeteroOS (introduced in section 3.1) and MMU-notifier based solution (proposed by reviewer A) capture A-bit hotness information by trapping every guest pagetable modification. That implies an expensive trap on every first access to a clean page. vTMM use PML hardware to track pagetable modifications, it only reduces the trap amount by a constant factor, but can not eliminate them. Making thing worse, for all these designs, further guest page table walks are also needed for translation to host-understandable addresses.

> HeteroOS, published in 2017, did not solve the crux of hotness tracking, instead they chose the most expensive software-based method, despite the availablility of hardware features including PEBS and PML, which are  introduced in [2008](https://github.com/torvalds/linux/commit/93fa7636dfdc059b25df148f230c0991096afdef) with Nehalem, and [2015](https://lore.kernel.org/all/1422413668-3509-1-git-send-email-kai.huang@linux.intel.com/) with Boardwell respectively. Their major contribution lies in delegating only page migraiton into the guest.

> Out of such insight we propose the core idea (thanks reviewer D for pointing out) of fully disaggregating data placement responsibilities—not only data migrations but most importantly hotness tracking—into guests from tiered memory provisioning in the host. We leverage PEBS that produce samples with ready-to-use guest virtual address that are directly written to buffer in guest memory. With routine buffer draining during context switches, we eliminate trapping or VMexits. Our identification algorithm operates in virtual address space, further eliminting the need for pagetable walks. The choice of virtual address also avoids the fragmentation issue often found in physical space caused by linux page allocator and map-on-first-touch policy which clobbers spatial locality.

> Another insight is that apart from low-cost tracking, accurate identification is another essence for making tiered mory performant, for which we demonstrate our range-based hotness identification, while migration only carries out the decision made during identificaiton. We are indeed inspired by DAMON\[Middleware'19\]\[HPDC'22\], we thank their contribution and appolgize for unintentionally leaving out due to space constraints.  However, DAMON is a region based memory profiling tools that are both inaccurate and requires user interaction. DAMON relies on a randomly selected page's A-bit access information to estimate the whole region's hotness from which that page is selected. Although DAMON also performs region split, such action is user configured, DAMON will only produce a user instructed number of ranges. Such design might be helpful for manual performance tunning, but is unable to serve as a full fledged automatic hotness identification solution and facilate data placement of a wide range of applications.

#### HeteroOS

We are very grateful to receive recognition from reviewers A, B, D, and E for our contribution as the first work to fully disaggregate data placement responsibilities—not only data migrations but also hotness tracking—into guests from tiered memory provisioning in the host. While HeteroOS exploits guest access information, hotness tracking is still performed in the host. As introduced in Section 3.1, this design choice, combined with their para-virtualization architecture, necessitates triggering a page fault on every guest page table modification to capture hotness information, severely hindering performance and eliminating the possibility of real-world adoption. As reviewer C aptly noted, emerging modern hardware motivates us to consider a novel responsibility assignment that leverages the newly available capabilities provided by PEBS to realize such disaggregation.

I am also deeply sorry for the negligence of limited evaluation and crowded figures due to the desire to elaborate our complete design in great detail and present our results across a wide range of applications.

