+++
+++

### Introduction v3.1

```
= Introduction
Efficient, elastic, scalable, and flexible virtual machine services form the backbone of today's virtualized cloud infrastructure, which directly powers customer-facing applications and enables higher-level cloud services such as serverless computing.
However, recent hardware trends threaten the continued evolution of these services.
While CPU core counts have increased tenfold (from 15 to 144 cores @intel8895v2 @intel6780e), the maximum memory per core has decreased to approximately one-third of its original capacity (from \~100GiB to \~28GiB per core @intel8895v2 @intel6780e), creating a critical memory scalability bottleneck.
Cloud vendors have responded by offering memory-optimized instances, but these come with reduced core counts, forcing an inefficient tradeoff.

Major cloud providers, including Azure @osdi24memstrata, Google @asplos23tmts, and Facebook @asplos22tmo, have turned to tiered memory architectures to address this scalability challenge.
These architectures supplement traditional DRAM as the fast memory tier (FMEM) with a second, slower memory tier (SMEM) using alternative media or interconnects.
Technologies such as Persistent Memory (PMEM) and Compute Express Link Memory Protocol (CXL.mem) can effectively double memory capacity at half the cost of additional DRAM @dell64dram @dell128pmem.
However, tiered memory architecture requires careful management to balance FMEM performance with SMEM capacity benefits.

An intuitive approach to extending tiered memory benefits to cloud tenants would be to incorporate existing hypervisor-based tiered memory management systems in the host machine.
However, existing tiered memory management systems for virtualized environments are all hypervisor-based, which incorporate access tracking and hotness management from within the host-side hypervisor.
Our experience with these approaches shown in @motivation-overhead reveals extreme overhead that is unacceptable for virtualized cloud environments where efficiency is paramount.
Furthermore, Processor Event-Based Sampling (PEBS), a superior hardware-based access tracking mechanism, demonstrated in state-of-the-art host-based systems @sosp23memtis @sosp21hemem, fundamentally cannot track guest memory accesses when deployed in virtualized hosts.
The widespread adoption of memory-optimized instances suggests that tenants are already aware of and accept performance-capacity tradeoffs similar to what tiered memory management systems address.
This tenant awareness, coupled with the incapacity of existing designs and the untapped potential of tiered memory management systems within the guest virtual machine itself, prompts us to rethink the traditional division of responsibilities in virtualization environments.

In this paper, we present a novel approach with guest-delegated tiered memory management (TMM).
We argue that the entire pipeline of TMM should be delegated to guests while hosts focus solely on making tiered memory available through tiered memory provisioning (TMP).
Our key insight is that while PEBS cannot be effectively utilized by hypervisor-based TMM solutions, it remains fully functional and is able to achieve high efficieny when approached properly from within the guest, a capability that has remained unexplored in tiered memory systems for virtualized environments.
Guest-delegated TMM leverages this guest-accessible PEBS to gather abundant access hotness information and utilizes locality information that primarily exists in the guest virtual address space.
This approach avoids the expensive and destructive scanning or inefficient sorting and LRU processing on uncorrelated pages employed by existing hypervisor-based solutions.

The cloud environment presents unique challenges to the design of tiered memory management systems with guest delegation.
The first challenge is ensuring elasticity and enabling cloud quality of service (QoS) management.
Memory resources in cloud environments are shared among tenants across diverse service tiers and are often overprovisioned @eurosys25hyperalloc.
Direct application of existing kernel-based solutions results in skewed provisioning of tiered memory, leading to resource wastage.
We address this challenge through a double balloon-based design called HyperFlex.

The second challenge lies in addressing the efficiency and scalability of guest-delegated TMM.
Cloud environments aim to minimize CPU overhead and rent all available CPU resources as virtual machines.
Directly repurposing kernel-based TMM inside guests would compound management overhead as the total number of virtual machines grows.
Each stage of the TMM pipeline must be carefully redesigned to maximize CPU resource efficiency and reduce compound overheads.

Our solution, HyperPlace, is the first to utilize Extended Page Table (EPT) friendly PEBS hardware as the hotness source in a virtualized environment, which is directly accessible by guests instead of traditional Translation Lookaside Buffer (TLB) flush intensive Page Table Entry Access (PTE.A) bits information.
This novel application of PEBS within guests represents a fundamental shift in how memory access patterns are tracked in virtualized environments.
The high-fidelity hotness data feeds into a range-based classifier operating in guest virtual address space, maximizing locality information undisturbed by both host and guest kernel memory allocators. 
Pages identified as hot or cold are symmetrically exchanged, further minimizing locking and TLB flush frequency.

We evaluate our designs across two tiered memory configurations: one with DRAM as fast memory and real PMEM as slow memory, and another with DRAM and emulated CXL.mem.
We test these configurations using seven real-world workloads spanning databases, scientific computing, graph processing, and machine learning. 
Our evaluation shows that leveraging the guest-delegated principle using HyperFlex as the TMP mechanism yields a direct performance improvement of 74% over traditional host-based solutions.
Incorporating HyperPlace as the guest-delegated TMM component improves real-world workload performance by up to 90% compared to the next best alternative.

In summary, we make the following contributions:
- We introduce the principle of guest-delegation for tiered memory management, which improves performance by up to 90% compared to existing solutions by leveraging guest-level access patterns and locality information while preserving guest flexibility to implement application-tailored management schemes.
- We pioneer the use of PEBS for memory access tracking in virtualized environments, demonstrating for the first time that this hardware feature can be effectively and efficiently utilized from within guest VMs for tiered memory management. //—a capability previously thought unworkable in virtualized systems.
- We design HyperFlex, a double balloon-based tiered memory provisioning mechanism that maintains cloud elasticity while enabling vendor-specific QoS control.
- We develop HyperPlace, a scalable guest TMM component that efficiently operates across multiple guests through our novel EPT-friendly PEBS sampling approach, range-based classification, and symmetric page exchange.
- We demonstrate the viability of our approach through comprehensive evaluation with seven real-world workloads across both DRAM-PMEM and DRAM-CXL.mem configurations.
```

### Introduction v3

```
= Introduction
Efficient, elastic, scalable, and flexible virtual machine services form the backbone of today's virtualized cloud infrastructure, which directly powers customer-facing applications and enables higher-level cloud services such as serverless computing.
However, recent hardware trends threaten the continued evolution of these services.
While CPU core counts have increased tenfold (from 15 to 144 cores @intel8895v2 @intel6780e), the maximum memory per core has decreased to approximately one-third of its original capacity (from \~100GiB to \~28GiB per core @intel8895v2 @intel6780e), creating a critical memory scalability bottleneck.
Cloud vendors have responded by offering memory-optimized instances, but these come with reduced core counts, forcing an inefficient tradeoff.

Major cloud providers, including Azure @osdi24memstrata, Google @asplos23tmts, and Facebook @asplos22tmo, have turned to tiered memory architectures to address this scalability challenge.
These architectures supplement traditional DRAM as the fast memory tier (FMEM) with a second, slower memory tier (SMEM) using alternative media or interconnects.
Technologies such as Persistent Memory (PMEM) and Compute Express Link Memory Protocol (CXL.mem) can effectively double memory capacity at half the cost of additional DRAM @dell64dram @dell128pmem.
However, tiered memory architecture requires careful management to balance FMEM performance with SMEM capacity benefits.

An intuitive approach to extending tiered memory benefits to cloud tenants would be to incorporate existing hypervisor-based tiered memory management systems in the host machine.
However, existing tiered memory management systems for virtualized environments are all hypervisor-based, which incorporate access tracking and hotness management from within the host-side hypervisor.
Our experience with these approaches shown in @motivation-overhead reveals extreme overhead that is unacceptable for virtualized cloud environments where efficiency is paramount.
Furthermore, Processor Event-Based Sampling (PEBS)—a superior hardware-based access tracking mechanism demonstrated in state-of-the-art host-based systems like @sosp23memtis and @sosp21hemem—fundamentally cannot track guest memory accesses when deployed in virtualized hosts.
The widespread adoption of memory-optimized instances suggests that tenants are already aware of and accept performance-capacity tradeoffs similar to what tiered memory management systems address.
This tenant awareness, coupled with the incapacity of existing designs and the untapped potential of tiered memory management systems within the guest virtual machine itself, prompts us to rethink the traditional division of responsibilities in virtualization environments.

In this paper, we present a novel approach with guest-delegated tiered memory management (TMM).
We argue that the entire pipeline of TMM should be delegated to guests while hosts focus solely on making tiered memory available through tiered memory provisioning (TMP).
Our key insight is that while PEBS cannot be effectively utilized by hypervisor-based TMM solutions, it remains fully functional and highly efficient when accessed directly from within the guest—a capability that has remained unexplored in tiered memory systems for virtualized environments.
Guest-delegated TMM leverages this guest-accessible PEBS to gather abundant access hotness information and utilizes locality information that primarily exists in the guest virtual address space.
This approach avoids the expensive and destructive scanning or inefficient sorting and LRU processing on uncorrelated pages employed by existing hypervisor-based solutions.

The cloud environment presents unique challenges to the design of tiered memory management systems with guest delegation.
The first challenge is ensuring elasticity and enabling cloud quality of service (QoS) management.
Memory resources in cloud environments are shared among tenants across diverse service tiers and are often overprovisioned @eurosys25hyperalloc.
Direct application of existing kernel-based solutions results in skewed provisioning of tiered memory, leading to resource wastage.
We address this challenge through a double balloon-based design called HyperFlex.

The second challenge lies in addressing the efficiency and scalability of guest-delegated TMM.
Cloud environments aim to minimize CPU overhead and rent all available CPU resources as virtual machines.
Directly repurposing kernel-based TMM inside guests would compound management overhead as the total number of virtual machines grows.
Each stage of the TMM pipeline must be carefully redesigned to maximize CPU resource efficiency and reduce compound overheads.

Our solution, HyperPlace, is the first to utilize Extended Page Table (EPT) friendly PEBS hardware as the hotness source in a virtualized environment, which is directly accessible by guests instead of traditional Translation Lookaside Buffer (TLB) flush intensive Page Table Entry Access (PTE.A) bits information.
This novel application of PEBS within guests represents a fundamental shift in how memory access patterns are tracked in virtualized environments.
The high-fidelity hotness data feeds into a range-based classifier operating in guest virtual address space, maximizing locality information undisturbed by both host and guest kernel memory allocators. 
Pages identified as hot or cold are symmetrically exchanged, further minimizing locking and TLB flush frequency.

We evaluate our designs across two tiered memory configurations: one with DRAM as fast memory and real PMEM as slow memory, and another with DRAM and emulated CXL.mem.
We test these configurations using seven real-world workloads spanning databases, scientific computing, graph processing, and machine learning. 
Our evaluation shows that leveraging the guest-delegated principle using HyperFlex as the TMP mechanism yields a direct performance improvement of 74% over traditional host-based solutions.
Incorporating HyperPlace as the guest-delegated TMM component improves real-world workload performance by up to 90% compared to the next best alternative.

In summary, we make the following contributions:
- We introduce the principle of guest-delegation for tiered memory management, which improves performance by up to 90% compared to existing solutions by leveraging guest-level access patterns and locality information while preserving guest flexibility to implement application-tailored management schemes.
- We pioneer the use of PEBS for memory access tracking in virtualized environments, demonstrating for the first time that this high-efficiency hardware feature can be effectively utilized from within guest VMs for tiered memory management. //—a capability previously thought unworkable in virtualized systems.
- We design HyperFlex, a double balloon-based tiered memory provisioning mechanism that maintains cloud elasticity while enabling vendor-specific QoS control.
- We develop HyperPlace, a scalable guest TMM component that efficiently operates across multiple guests through our novel EPT-friendly PEBS sampling approach, range-based classification, and symmetric page exchange.
- We demonstrate the viability of our approach through comprehensive evaluation with seven real-world workloads across both DRAM-PMEM and DRAM-CXL.mem configurations.
```

### Introduction v2

```
= Introduction
Efficient, elastic, scalable, and flexible virtual machine services form the backbone of today's virtualized cloud infrastructure, which directly powers customer-facing applications and enables higher-level cloud services such as serverless computing.
However, recent hardware trends threaten the continued evolution of these services.
While CPU core counts have increased tenfold (from 15 to 144 cores @intel8895v2 @intel6780e), the maximum memory per core has decreased to approximately one-third of its original capacity (from \~100GiB to \~28GiB per core @intel8895v2 @intel6780e), creating a critical memory scalability bottleneck.
Cloud vendors have responded by offering memory-optimized instances, but these come with reduced core counts, forcing an inefficient tradeoff.

Major cloud providers, including Azure @osdi24memstrata, Google @asplos23tmts, and Facebook @asplos22tmo, have turned to tiered memory architectures to address this scalability challenge.
These architectures supplement traditional DRAM as the fast memory tier (FMEM) with a second, slower memory tier (SMEM) using alternative media or interconnects.
Technologies such as Persistent Memory (PMEM) and Compute Express Link Memory Protocol (CXL.mem) can effectively double memory capacity at half the cost of additional DRAM @dell64dram @dell128pmem.
However, tiered memory architecture requires careful management to balance FMEM performance with SMEM capacity benefits.

An intuitive approach to extending tiered memory benefits to cloud tenants would be to incorporate existing host-based or hypervisor-based tiered memory management systems in the host machine.
However, our experience with these approaches reveals extreme overhead that is unacceptable for virtualized cloud environments where efficiency is paramount.
The widespread adoption of memory-optimized instances suggests that tenants are already aware of and accept performance-capacity tradeoffs similar to what tiered memory management systems address.
This tenant awareness, coupled with the possibility of applying emerging kernel-based tiered memory management systems within the guest virtual machine itself, prompts us to rethink the traditional division of responsibilities in virtualization environments.

In this paper, we present a novel approach with guest-delegated tiered memory management (TMM).
We argue that the entire pipeline of TMM should be delegated to guests while hosts focus solely on making tiered memory available through tiered memory provisioning (TMP).
Guest-delegated TMM leverages abundant access hotness information directly exported through guest mode hardware and utilizes locality information that primarily exists in the guest virtual address space.
This approach avoids the expensive and destructive scanning or inefficient sorting and LRU processing on uncorrelated pages employed by existing hypervisor-based solutions.

The cloud environment presents unique challenges to the design of tiered memory management systems with guest delegation.
The first challenge is ensuring elasticity and enabling cloud quality of service (QoS) management.
Memory resources in cloud environments are shared among tenants across diverse service tiers and are often overprovisioned. @eurosys25hyperalloc
Direct application of existing host-based solutions results in skewed provisioning of tiered memory, leading to resource wastage.
We address this challenge through a double balloon-based design called HyperFlex.

The second challenge lies in addressing the efficiency and scalability of guest-delegated TMM.
Cloud environments aim to minimize CPU overhead and rent all available CPU resources as virtual machines.
Directly repurposing host-based TMM inside guests would compound management overhead as the total number of virtual machines grows.
Each stage of the TMM pipeline must be carefully redesigned to maximize CPU resource efficiency and reduce compound overheads.

Our solution, HyperPlace, utilizes the efficient Extended Page Table (EPT) friendly Processor Event-Based Sampling (PEBS) hardware as the hotness source, which is directly accessible by guests instead of traditional Translation Lookaside Buffer (TLB) flush-intensive Page Table Entry Access (PTE.A) bits information.
This hotness data feeds into a range-based classifier operating in guest virtual address space, maximizing locality information undisturbed by both host and guest kernel memory allocators.
Pages identified as hot or cold are symmetrically exchanged, further minimizing locking and TLB flush frequency.

We evaluate our designs across two tiered memory configurations: one with DRAM as fast memory and real PMEM as slow memory, and another with DRAM and emulated CXL.mem.
We test these configurations using seven real-world workloads spanning databases, scientific computing, graph processing, and machine learning.
Our evaluation shows that leveraging the guest-delegated principle using HyperFlex as the TMP mechanism yields a direct performance improvement of 74% over traditional host-based solutions.
Incorporating HyperPlace as the guest-delegated TMM component improves real-world workload performance by up to 90% compared to the next best alternative.

In summary, we make the following contributions:

- We introduce the principle of guest-delegation for tiered memory management, which improves performance by up to 90% compared to existing solutions by leveraging guest-level access patterns and locality information while preserving guest flexibility to implement application-tailored management schemes.
- We design HyperFlex, a double balloon-based tiered memory provisioning mechanism that maintains cloud elasticity while enabling vendor-specific QoS control.
- We develop HyperPlace, a scalable guest TMM component that efficiently operates across multiple guests through EPT-friendly PEBS sampling, range-based classification, and symmetric page exchange.
- We demonstrate the viability of our approach through comprehensive evaluation with seven real-world workloads across both DRAM-PMEM and DRAM-CXL.mem configurations.
```

### Introduction v1

```
= Introduction
// Efficient, elastic, scalable and flexible virtual machine service is at the backbone of today's virtualized cloud service, which is direcly exposed to customers as well as siently powers serverless computing.
// However, hardware trends has began to threaden the continue offering of such services.
// CPU core counts have grown tenfold (from 15 to 144 @intel8895v2 @intel6780e), yet maximum possible memory per CPU core has dropped to about one-third of its original capacity (from \~100GiB to \~28GiB @intel8895v2 @intel6780e), limiting future scalability.
// Cloud vendors have resorted to offer memory-optimized instances which despite has higher memory capacity but has core counts cut in half.
Efficient, elastic, scalable, and flexible virtual machine services form the backbone of today's virtualized cloud infrastructure, which is both directly exposed to customers and silently powers other cloud services, such as serverless computing.
However, recent hardware trends have begun to threaten the continued offering of such services.
While CPU core counts have grown tenfold (from 15 to 144 @intel8895v2 @intel6780e), the maximum possible memory per CPU core has dropped to approximately one-third of its original capacity (from \~100GiB to \~28GiB @intel8895v2 @intel6780e), limiting future scalability. 
Cloud vendors have responded by offering memory-optimized instances, which provide higher memory capacity but with core counts cut in half.

// Major cloud adopters, such as Azure @osdi24memstrata, Google @asplos23tmts, and Facebook @asplos22tmo, have turned to tiered memory architectures to expand memory capacity.
// In additional to DRAM as the fast memory tier (FMEM), a second slow memory tier (SMEM) of alternative media or interconnects has been added.
// Technologies like Persistent Memory (PMEM) and Compute Express Link Memory Protocol (CXL.mem) can double memory at half the cost of additional DRAM @dell64dram @dell128pmem.
// However, tiered memory architecture requires careful management to secure the best of the FMEM performance and SMEM capacity.
Major cloud providers, including Azure @osdi24memstrata, Google @asplos23tmts, and Facebook @asplos22tmo, have turned to tiered memory architectures to expand memory capacity.
In addition to DRAM as the fast memory tier (FMEM), a second slow memory tier (SMEM) using alternative media or interconnects has been introduced.
Technologies such as Persistent Memory (PMEM) and Compute Express Link Memory Protocol (CXL.mem) can effectively double memory capacity at half the cost of additional DRAM @dell64dram @dell128pmem.
However, tiered memory architecture requires careful management to balance FMEM performance with SMEM capacity benefits.


// An intutive way to make the benefits of tiered memory avaialbe to cloud tenantes would be incooporating existing host-based or hypervisor-based tiered memory management systems in the host machine and expose a unified large memory space to the guest virtual machines.
// However, the wide adoption of existing memory-optimized instances imply tenants' awareness and acceptance of a performance and capacity tradeoff similar to what tiered memory management systems faces.
// Such implication together with the possibility of applying blooming kernel-based tiered memory management systems not in host but within the guest virtual machine itself let us question and rethink the traditional responsibility division in virtualization environment.
An intuitive approach to making tiered memory benefits available to cloud tenants would be incorporating existing host-based or hypervisor-based tiered memory management systems in the host machine.
However, our experience with them  in @motivation-overhead shows extreme overhead which is unbearable for virtualized cloud's efficiency need.
In such moments of hesitation, the widespread adoption of memory-optimized instances suggests tenants' awareness and acceptance of a performance-capacity tradeoff similar to what tiered memory management systems address.
This tenant awareness, coupled with the possibility and felxibility of applying emerging kernel-based tiered memory management systems within the guest virtual machine itself, prompts us to question and rethink the traditional division of responsibilities in virtualization environments.

In this paper, we present a novel approach with guest-delegated tiered memory management (TMM).
We argue that the entire pipeline of TMM should be delegated to guests while hosts focus on making tiered memory available to guests through tiered memory provisioning (TMP).
Guest-delegated TMM is able to leverage abundant access hotness information directly exported through guest mode hardware and make use of locality information that mostly exists in guest virtual address space, instead of relying on expensive and desctructive scanning or barbaric sorting and LRUing on uncorrelated pages which are incooprated by existing hypervisor-based solutions.

However, the cloud environments present unique challenges to the design of a tiered memory mangement systems with guest delegation.
The first challenage is how to ensure elasticity and enable cloud quality of service (QoS) management.
Memory resource in cloud is shared among tenants across diverse service tiers and are often overprovisioned. @eurosys25hyperalloc
Direct application of existing host-based solution results in a skewed provision of tiered memory which leads to resource wastage.
We address the first challenge through a double balloon based design called HyperFlex.

The second challenage lies in addressing efficiency and scalability of guest-delegated TMM.
Cloud environments target for minimizing CPU overhead and renting all available CPU resources as virtual machines.
A direct repurpose of host-based TMM inside guests as guest-delegated TMM would compound the management overhead as total number of virtual machines grows.
A careful redesign of each stage of the TMM pipeline needs to be reconsidered to extract maximum CPU resources and lowering the compound overheads.

We propose the design of HyperPlace.
HyperPlace makes use of the efficient EPT-friendly PEBS hardware as the hotness source that is directly accessible by guests instead of traditional TLB flush intensive PTE.A bits information.
Such hotness souce is fed to a range-based classifier operating in guest virtual address space which maximizing locality information undisturbed by both host and guest kernel memory allocator.
Pages identified as hot and cold are symmetrically exchanged futher minimizing locking and TLB flush frequency.

We evaluate our designs across two tiered memory configurations: one with DRAM as fast memory and real PMEM as slow memory, and another with DRAM and emulated CXL.mem.
We test these configurations using seven real-world workloads spanning databases, scientific computing, graph processing, and machine learning.
Evaluation shows that leveraging guest delegated pricinple using HyperFlex as the TMP mechanism yields a direct performance improvement over traditional host-based solution by 74%.
Incooperting HyperFlex as the guest-delegated TMM component improves real-world workload performance by up to 90% compared to the next best alternative.

In summary, we make the following contributions.
- We propose the design principle of guest-delegation.
  Delegating TMM to guests makes tiered memory efficient and scalable through leveraging easy-obtainable access and locality information.
  It also leave guests with the flexibility of choosing their own application-tailored management scheme.
  Leaving only TMP in hosts, perseves elasticity and enables vendor-sepcific QoS control.
- We design a double balloon-based HyperFlex TMP solution that ensure cloud elasticity.
- We design an scalable HyperPlace guest TMM component efficiently runnint concurrently under multiple guests leveraging EPT-friendly PEBS sampling, range-based clasisfication and symmetric page exchange.
- We demonstrate the avaialblity and superiority of PEBS-based hotness tracking in virtualized environment.
- We evaluation our TMP and TMM design under both real-world and emulated tiered memory environment.
```
