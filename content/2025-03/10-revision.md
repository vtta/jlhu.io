revision写作的主要idea还是围绕hypervisor-based perormance issues:

> The core insight behind our work is that existing hypervisor-based hotness tracking methods impose significant performance penalties. HeteroVisor/HeteroOS (Section 3.1) and MMU-notifier solutions (proposed by Reviewer A) capture A-bit hotness information by trapping every guest pagetable modification, causing expensive traps on first access to clean pages. While vTMM uses PML hardware to track pagetable modifications, it only reduces trap frequency by a constant factor without eliminating them. All these designs require further guest page table walks to translate addresses to host-understandable ones.

虚拟化TM总共有三个地方可以做管理: host kernel, hypervisor, guest kernel. 我们首先需要排除host kernel. 然后着重讲hypervisor为什么不好. 最后引出我们guest-based的solution.

Host kernel的主要问题是没有VM的概念. 在其中做migration会导致各个VM内存分布不均. 

Hypervisor的主要问题是performance. 依照我们rebuttal的分析, 主要为两点: 1) hotness tracking; 2) address translation; 我们可以通过简单的microbenchmark来demo. 关于tracking, 我们可以通过展示通过MMU-notifer以及PML两种方式来分别展示HeteroVisor/HeteroOS以及vTMM两派解决方法. 



目前的入口是KVM里面的函数: `kvm_mmu_notifier_test_young`以及`kvm_test_age_gfn`.





看看EPT中的A-bit相关描述.