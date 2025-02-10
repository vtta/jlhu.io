+++
+++
## Overview

> PPT page 2:
>
> Executive Summary
>
> Background:
>
> - CPU **core counts scaling faster** than memory capacity
>
> - CXL enables **second-tier memory** to facilitate core scaling
>
> - But CXL adds latency that hurts performance if not mitigated
>
> - Software tiering helps some but **is not well suited for public clouds**
>
> Contributions:
>
> - Intel Flat Memory Mode: First **hardware-managed** memory tiering for CXL
>
> - But still has **limitations** that degrade workloads
>
> - Memstrata: Memory allocator for hardware tiering to **mitigate outliers**
>
> - Slowdown reduces to ~5% vs. unattainable one-tier memory

Memstrata自己的定位是memory allocator. 管理的单位是VM, 即确定每个VM应该分配多少DRAM以及CXL.mem. Memstrata检测每个VM的slowdown, 通过多给slowdown更严重的VM分配DRAM达到performance isolation的目的. 宏观指标是slowdown, 但是由于无法测量, 遂使用微观指标L3 miss来估计slowdown.

这里的假设就是每个VM使用不同的workload, 而我们考虑的场景中每个VM都是一样的.

## Abstract

> Cloud providers are reluctant to deploy software-based tiering due to **high overheads** in virtualized environments. Hardware-based memory tiering could place data at cacheline granularity, mitigating these drawbacks. However, hardware is oblivious to application-level performance.

这里并没有讲出高开销的来源. 但是后面暗示了原因之一可能是由于不能做到cacheline粒度的管理. 除此之外软件管理相对硬件方法也有好处, 那就是可以看到应用程序的具体性能.

> We propose combining hardware-managed tiering with software-managed performance isolation to overcome the pitfalls of either approach. We introduce **Intel ®Flat Memory Mode**, the ﬁrst hardware-managed tiering system for CXL.

结合硬件的低开销和软件对应用性能的了解, 提出“硬件管层间放置, 软件管性能隔离”的方法.

这里可以攻击Flat Memory Mode, 仅仅是Intel支持. 对比“instruction sampling”, Intel和AMD都有自己的实现: PEBS/IBS. 用了FMM反而会vendor lock-in.

> Our evaluation on a full-system prototype demonstrates that it provides performance close to regular DRAM, with **no more than 5% degradation for more than 82% of workloads.**

这里的性能提升靠的是找足够多对PM不敏感的应用场景. 比如Fig 5中, 其实很多(尤其是web和spark)应用在纯CXL下slowdown就<5%. 这里应该固定<5%slowdown, 比CXL-only和Memstrata到底分别有多少workload小于这个值.

> Despite such small slowdowns, we identify two challenges that can still degrade performance by up to 34% for “outlier” workloads: (1) memory contention across tenants, and (2) intra-tenant contention due to conﬂicting access patterns.

这里(1)就非常主观了, contention如何定义? 如何保证contention能反应真实的情况?

这里也可以给出我们不做多vm的根据, 即没有数据. 我们无法在测试环境中有代表性的复现生产环境出现的workload更别提他们之间的contention.

> To address these challenges, we introduce Memstrata, a lightweight multi-tenant **memory allocator**.

这里他自己吧自己定义为allocator, 直接和我们不在一个维度上. 我们同时考虑了hypervisor和guest, 是一个完整的solution.

> Memstrata employs page coloring to eliminate inter-VM contention. It improves performance for VMs with access patterns that are sensitive to hardware tiering by allocating them more local DRAM using an online slowdown estimator.

主要的贡献也很简单, 就是给对内存速度敏感的应用多分配DRAM. 这个和我们非常orthogonal. 我们是考虑如何在一个fixed budget的vm里放置应用内存, 而Memstrata是考虑不同vm间怎么分配不同内存. Memstrata分配更多内存之后呢? 仍然需要一种方法来更好的使用这些内存.

> In multi-VM experiments on prototype hardware, Memstrata is able to **identify performance outliers** and **reduce their degradation from above 30% to below 6%**, providing consistent performance across a wide range of workloads.

30->6%的减少是at the cost of what? 假设没有足够的DRAM供Memstrata去额外分配的时候怎么办?

## Introduction

> In public clouds, virtual machine (VM) memory sizes are increasing, with typical conﬁgurations of 4–32GB per virtual CPU [6, 7, 12]. However, the DRAM capacity accessible via DDR channels is lagging the rapid growth in available cores, due to physical limitations associated with scaling the capacity of DDR DIMMs [73,91,92].
>

这里可以为我们的configuration做支撑, i.e. 4T16G.

> Most prior work on memory tiering assumes software (e.g., the hypervisor or the OS) has full control over data placement, ... We term this software-managed memory tiering.

这里的software/hardware区分仅依赖于谁来管placement. Tracking环节不考虑在内.

> Section I, paragraph 4:
>
> However, in our experience at Microsoft Azure, these approaches face severe limitations in virtualized environments (§2). For example, **instruction sampling is not supported for VMs** and has **privacy implications**. Fine-grained page table operations consume excessive host CPU cycles [44,79]. In addition, with software-managed tiering, the hypervisor/OS can only manage memory at page granularity. This leads to suboptimal decisions [76] for the common case where a mix of hot and cold data resides on the same page. This is particularly problematic for hypervisors that use larger page sizes (e.g., 2 MB and 1 GB) to reduce overheads. All of these drawbacks make deploying software-based memory tiering techniques unattractive in general-purpose cloud environments.

第一句: Memstrata的观点仅限于Azure, 也就是其他云考虑不考虑在vm内部使用IS并没有一个定论. 其中一个反例是目前可以在[lkml中](https://lore.kernel.org/lkml/20221206082944.59837-1-likexu@tencent.com/)看到Tencent有涉及在vm中启用IS的讨论.

### Instruction sampling in VMs

在[Intel的SDM](https://web.archive.org/web/20240816185003/https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) Volumn 3B, section [20.9.5](https://web.archive.org/web/20240315003216/https://cdrdv2-public.intel.com/812388/253669-sdm-vol-3b.pdf)中讲到了EPT-friendly PEBS.

> > 20.9.5 EPT-Friendly PEBS
> >
> > The 3rd generation Intel Xeon Scalable Family of processors based on Ice Lake microarchitecture (and later processors) and the 12th generation Intel Core processor (and later processors) support VMX guest use of PEBS when the DS Area (including the PEBS Buffer and DS Management Area) is allocated from a paged pool of EPT pages. In such a configuration PEBS DS Area accesses may result in VM exits (e.g., EPT violations due to “lazy” EPT page-table entry propagation), and in such cases the PEBS record will not be lost but instead will “skid” to after the subsequent VM Entry back to the guest. For precise events the guest will observe that the record skid by one event occurrence, while for non-precise events the record will skid by one instruction.

Guest PEBS从Ice Lake开始支持. 前提是DebugStore, 包括PEBS buffer都需要从guest physical memory中分配. 正常情况CPU写入PEBS record时会尊重EPT.

这里有疑问:

- 什么是paged-pool和non-paged pool?
- EPT pages指代的到底是什么?

Section 20.3.1.1.1,  Programming PEBS Facility, paragraph 2中有提到:

> > **The recording of PEBS records may not operate properly if accesses to the linear addresses in the DS buffer management area or in the PEBS buffer (see below) cause page faults, VM exits, or the setting of accessed or dirty flags in the paging structures (ordinary or EPT)**. For that reason, system software should establish paging structures (both ordinary and EPT) to prevent such occurrences. Implications of this may be that an operating system should allocate this memory from a non-paged pool and that system software cannot do “lazy” page-table entry propagation for these pages. A virtual-machine monitor may choose to allow use of PEBS by guest software only if EPT maps all guest-physical memory as present and read/write.

这里(non-)paged pool的概念均和paging即page table mapping相关. non-paged pool就是已经提前建立好页表映射的(虚拟)页; 而paged pool就是并没有提前建立好页表映射, 而是依赖于page fault handler去on-demand的分配物理内存的页.

从这段话也可以读出并不是Ice Lake开始才能支持Guest PEBS, 只是在这之前PEBS相关内存的写入不能触发page fault. 解决方法就是提前建立这些地址的页表映射. 在Section 18.4.9.2中也有提到相关内容(原文如下). 对于Ice Lake之前的Guest PEBS可以在保证所有guest physical memory都被映射且有可读可写权限下启用. (因为host并不知道guest会把DS分配在哪个位置. 为了确保DS有完整的page table mapping, 必须让所有的GPA都有mapping.)

> > 18.4.9.2 Setting Up the DS Save Area
> >
> > ...
> >
> > Some newer processor generations support “lazy” EPT page-table entry propagation for PEBS; see Section 20.3.9.1 and Section 20.9.5 for more information. A virtual-machine monitor may choose to allow use of PEBS by guest software only if EPT maps all guest-physical memory as present and read/write.
>
>
>
> > 20.3.9.1 Processor Event Based Sampling (PEBS) Facility
> >
> > ...
> >
> > The 3rd generation Intel Xeon Scalable Family of processors based on the Ice Lake microarchitecture introduce EPT-friendly PEBS. This allows EPT violations and other VM Exits to be taken on PEBS accesses to the DS Area.

AMD的PEBS叫Instruction-Based Sampling. 目前只找到#47414中有关于IBS的详细介绍, 但是已经年代久远. The only detailed description of AMD IBS we can find is from year 2014 showing AMD family 0x15. We are unable to find sufficient architecture documentation containing the description of IBS of the recent AMD family 0x17/0x19/0x1a. We are also unable to get hand on the AMD hardware to carry out experiemnts.

#### 总结一下

- 支持是肯定支持
  - Ice Lake之前支持比较原始, PEBS buffer不能on-demand paging. 可以用cooperative的方式在启用PEBS之前通知hypervisor来建立映射.
  - Ice Lake之后做到了完全无感(EPT-friendly)的guest pebs.
- 技术细节
  - PMU记录用户设定的事件, 每隔给定的出现次数后生成一个sample, 并写入到PEBS buffer.
  - 不支持的理由主要是PEBS buffer写入过程中不能触发page fault以及V M exit. (Intel SDM Section 20.3.1.1.1) 这里主要是architectural defect.

On the contrary to what is reported in the prior work[^1], we find that instruction sampling is well-supported in both legacy and modern Intel architecture. In Intel's instruction sampling, PEBS, the PMU records architectural events as programmed by the user. After a given occurrence of that event, a sample is generated and is logged by the PMU to a buffer in memory called PEBS buffer. In  legacy architectures that predate Ice Lake, there is an architectural defect causing the CPU core to malfunction when a write to the PEBS buffer triggers page faults or VM-exits. Due to this defect, it's believed that instruction sampling is not supported in a VM. However, we find that is not true for both the legacy and modern architecture. Intel suggests [^2] that for legacy architecture after mapping all guest pages as present and read-writable, the hypervisor could allow guests to enable PEBS. This is both in optimal and unnecessary. Mapping all pages as present trades off the ability to overcommit memory. Modifing the perf subsystem to cooperatively let the hypervisor pre-fill the mapping when enabling PEBS in the guests is enough. Starting from Ice Lake, Intel has introduced a patch called EPT-friendly PEBS. In this patch, the access to PEBS buffer could trigger page faults or VM-exits, allowing hypervisors to enable guest PEBS by default.

Besides x86-64, RISC-V also supports instruction sampling in guests [^3] [^4].

[^1]: [Managing Memory Tiers with CXL in Virtualized Environments - OSDI 2024](https://www.usenix.org/conference/osdi24/presentation/zhong-yuhong)

[^2]: [Section 20.3.1.1.1 - Intel® 64 and IA-32 Architectures Software Developer Manuals](https://web.archive.org/web/20240816185003/https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
[^3]: [RISC-V Perf Tool Status - 2019 RISC-V Workshop Taiwan](https://web.archive.org/web/20221205152220/https://riscv.org/wp-content/uploads/2019/03/10.05-Mar-13-Alan-Kao-Per-Tool-Status-RISCV-workshop.pdf)
[^4]: [RISC-V SBI v2.0 PMU improvements and Perf sampling in KVM guest](https://lore.kernel.org/kvm/20240416184421.3693802-1-atishp@rivosinc.com/)

### Privacy

隐私问题的根源是PEBS会把地址信息记录到PEBS buffer. 对于非虚拟化环境, PEBS buffer是全局共享的(仅以目前Linux的perf event subsystem实现为准). 假若有恶意用户掌握了PEBS buffer, 则他可以嗅探全部在使用PEBS的进程的访存信息. 但是在虚拟化环境下, 每个vm的PEBS buffer都是独立且on-demand分配的. 即使有某个vm的PEBS buffer被恶意用户掌握, 其他vm的PEBS buffer仍然是隔离状态且不受影响. 只有当恶意用户攻入`IA32_DS_AREA`寄存器的模拟逻辑, 进而看到所有vm的PEBS buffer, 其才会看到全局的地址信息. 但是侵入kvm子系统内部的寄存器模拟逻辑已经让恶意用户拥有了访问全局物理内存的能力, 所以弱点并不在PEBS. 如果需要为privacy提供足够的支持, 用户可以考虑将vm放进独立的SGX enclave. SGX enclave可以阻止任何enclave外的非SGX指令访存[^2.5.1]

- [ ] 讨论SGX与PEBS的关系.
  - [kernel](https://github.com/torvalds/linux/commit/3c0c2ad1ae75963c05bf89ec91918c6a53a72696) [manual](https://www.intel.com/content/dam/develop/external/us/en/documents/329298-002-629101.pdf)
  - 目前看了SGX和PEBS是orthogonal的. SGX保护vCPU进程以及VM内存, PEBS buffer的读写均来自SGX内部, 所以SGX不影响PEBS的支持.

[^2.5.1]: [Section 2.5.1 Access-control for Accesses that Originate from non-SGX Instructions - Intel Software Guard Extensions Programming Reference](https://www.intel.com/content/dam/develop/external/us/en/documents/329298-002-629101.pdf)
