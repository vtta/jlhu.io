+++
+++

[[storyline]](https://www.notion.so/D5-1-Tiered-Memory-Management-for-VM-a157aba5f8894e2bbd83aeb60f861d31) [[intro]](https://www.notion.so/Intro-1c3c2297ae2680f1bb55ea7e7b6d8653)

## Delegating Tiered Memory Management for Virtualized Clouds with HyperTier

---

### Introduction v2

- Cloud faces memory capacity expansion challenges
  - Efficient + elastic + scalable VM is the backbone of today's cloud service
  - memory per CPU trend
- TM briges this gap but requires careful management
- Cloud already presents capacity and performance tradeoff to guests
  - Mem optimized VM instances
  - Blooming OS based TMM support
  - Rethink responsibilities division

- A novel angle with guest-delegated TMM
  -

### Introduction (围绕delegation单线讲: 提出delegation->优势->遇到的问题->解决方案)

- VM is at the heart of cloud datacenter
  - Basic service unit
  - Requirements: efficiency + elasticity + scalability
- Industry's goal in using TM and motivation
  - Expand capacity
  - Reduce TCO

- Careful TMM (tracking/classification/migration) is required to leverage TM
  - TM characteristic: fast/small + slow/large
  - frequent accessed -> FMEM

- ~~SOTA are all from host/hypervisor level  (shorter)~~
  - ~~HeteroVisor/OS: trapping to collect guest A-bit info~~
  - ~~Reuse host-based with MMU notifier backend: collect A-bit from EPT~~
  - ~~HeMem/Memtis: PEBS in host cannot collect guest accesses~~
  - ~~insight: A-bits require TLB flushes  (->overhead)~~
  - ~~insight: lack of easy accessible hotness information (->PEBS)~~
- A novel angle with guest-delegated TMM
  - Host/hypervisor-level mgmt exhibits server mgmt overhead
  - No longer require TLB flushes; utilize easily accessible PEBS hotness
  - ~~How (responsibility)? Leave only provision in host (elasiticity) and enables QoS control~~
  - Delegate hotness mgmt to guest (light weight mgmt; scalability)

- Challenges: how to ensure efficienty and scalability
  - Tracking: readily available PEBS hotness and eliminate TLB flushes
  - Classification: utilize locality available only in guest
  - Page-exchange: reduce metadata ops (locking; flushes)
- Contribution
  - Proposal of guest-delegation
  - Guest components
  - Provision
  - Evaluation
  - Open source

---

### Background

- (Foundational Knowledge) TMM and its pipeline
- (Prior research) Hypervisor-based TMM
- (Knowledge gap) EPT-friendy PEBS

### Motivation

- Hypervisor-based TMM overhead
-

---

- [E2E principle](http://web.mit.edu/Saltzer/www/publications/endtoend/endtoend.pdf)

