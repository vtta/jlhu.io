+++
+++

## Selected

| Relvance | Conf | Track | Title | Comment |
| -------- | ---- | ----- | ----- | ------- |
| √√√ | ATC23 | Serverless | [On-demand Container Loading in AWS Lambda](https://www.usenix.org/conference/atc23/presentation/brooker) | 落地方案 |
| √√√  | SOSP24 | Serverless | [TrEnv: Transparently Share Serverless Execution Environments Across Different Functions and Nodes](https://doi.org/10.1145/3694715.3695967) | Enable sharing of remote memory |
| √√√ | ASPLOS24 | Serverless | [FaaSMem: Improving Memory Efficiency of Serverless Computing with Memory Pool Architecture](https://doi.org/10.1145/3620666.3651355) | Memory segment-wise offloading of idle containers to remote memory pool |

### [ASPLOS24]FaaSMem

核心思路是把idle container的部份内存offload到remote memory. 此文将offload的贡献归功于tierd memory system, 并以TMO作为SOTA以及对比对象[^TMO]. 此文的贡献在于能达到更好的QoS.

[^TMO]:其思路是引入内存压力让kernel换页到storage.

#### Discussion

他是基于container的, 我们基于vm可以不比较. 如果我们要对比的话我们性能应该更优, 因为此文sec7中提到其也采用了swap的offload方式.

此文展示贡献的方式需要通过trace来区分cold/hot.

此文的实现方式也比较讨巧, 其extend了MGLRU, 用额外的generation代表了bucket.

### [SOSP24]TrEnv

核心贡献: reboot->restore->**repurpose**

核心思路还是sharing, 尽量抽象出Fn之间的相同部份. 但是这个和remote memory怎么搭上? TrEnv将snapshot细分为repurposable sandbox以及linux process. 通俗来讲实际对应了system state以及user state. system state包括container的IO device以及isolation state, user state则包括了runtime以及application state. SS可以在Fn间复用, 故名“repurposable”, 从而减少了cold boot次数. 而US, 此文称为mm-template相比全量内存镜像则更小, 进而减小了存储压力, 并适合放在remote memory.

US的恢复则需要OS的支持, 本质是CoW paging. 对于CXL, 可以直接读取RO页, RW页采取CoW paging. 对于RDMA, RO/RW均触发major pagefault.

这里仍然是contianer-based solution, 如果改用VM, 能否继续share SS?

理论上来说是可以, 但是其实难点在于US的替换. 对于container, 我们只需要替换为一个新的process然后mmap准备好虚拟内存从`_start`开始执行就行了. 但是对于一个VM, 在外面只能看到整个物理内存, 所有的SS和US都混合在了一起. 想要恢复US则需要在ABI层级的接口上做文章.

先形成一个文章大纲, 后续是否继续实现再根据难度和时间决定.

更重要的问题是这么做的motivation是什么?

### [ATC24]PASS

## Full

Any recent major conference papers with keyword **serverless** or **microservice** are listed below:

| Relvance | Conf      | Track                                      | Title                                                        | Comment                                                      |
| -------- | --------- | ------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| √        | EuroSys25 | ???                                        | [Serverless Cold Starts and Where to Find Them](https://arxiv.org/pdf/2410.06145) | Huawei trace                                                 |
| √        | OSDI24    | Memory Management                          | [Sabre: Hardware-Accelerated Snapshot Compression for Serverless MicroVMs](https://www.usenix.org/conference/osdi24/presentation/lazarev) | Improve snapshot prefetch coverage with hw compression to extend to reduce cold-startup time |
| xxx      | OSDI24    | Low-Latency LLM Serving                    | [ServerlessLLM: Low-Latency Serverless Inference for Large Language Models](https://www.usenix.org/conference/osdi24/presentation/fu) | don't care                                                   |
| x        | SOSP24    | Serverless                                 | [Dirigent: Lightweight Serverless Orchestration](https://doi.org/10.1145/3694715.3695966) | Reduce cluster-wide Fn scheduling delay                      |
| x        | SOSP24    | Serverless                                 | [Unifying Serverless and Microservice Workloads with SigmaOS](https://doi.org/10.1145/3694715.3695947) | OS designed with serverless in-mind                          |
| xxx      | SOSP24    | Serverless                                 | [Caribou: Fine-Grained Geospatial Shifting of Serverless Applications for Sustainability](https://doi.org/10.1145/3694715.3695954) | 将serverless迁移到低碳区域以减少碳排放                       |
| √√√      | SOSP24    | Serverless                                 | [TrEnv: Transparently Share Serverless Execution Environments Across Different Functions and Nodes](https://doi.org/10.1145/3694715.3695967) | Enable sharing of remote memory                              |
| √        | EuroSys24 | Networks                                   | [Serialization/Deserialization-free State Transfer in Serverless Workflows](https://doi.org/10.1145/3627703.3629568) | Optimize state transfer via remote memory map                |
| √        | EuroSys24 | BFT, Serverless, Cloud and Edge            | [Pronghorn: Effective Checkpoint Orchestration for Serverless Hot-Starts](https://doi.org/10.1145/3627703.3629556) | Choose a better snapshot timing                              |
| √        | EuroSys24 | Systems for ML                             | [Optimus: Warming Serverless ML Inference via Inter-Function Model Transformation](https://doi.org/10.1145/3627703.3629567) | Start a ML inference container from a warm but idle one      |
| xxx      | EuroSys24 | Scheduling, Shifting and Scaling Workloads | [Atlas: Hybrid Cloud Migration Advisor for Interactive Microservices](https://doi.org/10.1145/3627703.3629587) | A learnt method to offload microservies to worse infra       |
| xxx      | EuroSys24 | Scheduling, Shifting and Scaling Workloads | [Erlang: Application-Aware Autoscaling for Cloud Microservices](https://doi.org/10.1145/3627703.3650084) |                                                              |
| xxx      | ASPLOS24  | Serverless                                 | [λFS: A Scalable and Elastic Distributed File System Metadata Service using Serverless Functions](https://doi.org/10.1145/3623278.3624765) |                                                              |
| √√√      | ASPLOS24  | Serverless                                 | [FaaSMem: Improving Memory Efficiency of Serverless Computing with Memory Pool Architecture](https://doi.org/10.1145/3620666.3651355) | Memory segment-wise offloading of idle containers to remote memory pool |
| √        | ASPLOS24  | Serverless                                 | [CodeCrunch: Improving Serverless Performance via Function Compression and Cost-Aware Warmup Location Optimization](https://doi.org/10.1145/3617232.3624866) | Use compression to optimize cold startup time                |
| √        | ASPLOS24  | Serverless                                 | [RainbowCake: Mitigating Cold-starts in Serverless with Layer-wise Container Caching and Sharing](https://doi.org/10.1145/3617232.3624871) | Space and startup tradeoff                                   |
| x        | ASPLOS24  | Serverless                                 | [Flame: A Centralized Cache Controller for Serverless Computing](https://dl.acm.org/doi/10.1145/3623278.3624769) | Better cache decision                                        |
| xxx      | ASPLOS24  | Dynamic Analysis and Instrumentation       | ShapleyIQ: Influence Quantification by Shapley Values for Performance Debugging of Microservices |                                                              |
| xxx      | ASPLOS24  | Graph Neural Networks                      | Sleuth: A Trace-Based Root Cause Analysis System for Large-Scale Microservices with Graph Neural Networks |                                                              |
|          |           |                                            |                                                              |                                                              |
|          |           |                                            |                                                              |                                                              |
|          |           |                                            |                                                              |                                                              |
|          | OSDI23    | Store Your Bits                            | [No Provisioned Concurrency: Fast RDMA-codesigned Remote Fork for Serverless Computing](https://www.usenix.org/conference/osdi23/presentation/wei-rdma) | A fast remote fork over RDMA                                 |
|          | OSDI23    | Verify Your Bits                           | [Automated Verification of Idempotence for Stateful Serverless Applications](https://www.usenix.org/conference/osdi23/presentation/ding) |                                                              |
|          | SOSP23    | Cloud                                      | [XFaaS: Hyperscale and Low Cost Serverless Functions at Meta](https://dl.acm.org/doi/10.1145/3600006.3613155) |                                                              |
|          | SOSP23    | Distributed systems                        | [Halfmoon: Log-Optimal Fault-Tolerant Stateful Serverless Computing](https://dl.acm.org/doi/10.1145/3600006.3613154) |                                                              |
|          | EuroSys23 | Serverless                                 | Palette Load Balancing: Locality Hints for Serverless Functions |                                                              |
|          | EuroSys23 | Serverless                                 | With Great Freedom Comes Great Opportunity: Rethinking Resource Allocation for Serverless Functions |                                                              |
|          | EuroSys23 | Serverless                                 | Groundhog: Efficient Request Isolation in FaaS               |                                                              |
|          | ASPLOS23  | Clouds                                     | AQUATOPE: QoS-and-Uncertainty-Aware Resource Management for Multi-stage Serverless Workflows |                                                              |
|          | ASPLOS23  | Clouds                                     | [Erms: Efficient Resource Management for Shared Microservices with SLA Guarantees](https://dl.acm.org/doi/10.1145/3567955.3567964) |                                                              |
|          | ASPLOS23  | Clouds                                     | [Snape: Reliable and Low-Cost Computing with Mixture of Spot and On-demand VMs](https://doi.org/10.1145/3582016.3582028) |                                                              |
| xxx      | ASPLOS23  | Deep Learning                              | ElasticFlow: An Elastic Serverless Training Platform for Distributed Deep Learning |                                                              |
| √√√      | ATC23     | Serverless                                 | [On-demand Container Loading in AWS Lambda](https://www.usenix.org/conference/atc23/presentation/brooker) |                                                              |
|          |           |                                            |                                                              |                                                              |
|          |           |                                            |                                                              |                                                              |
|          |           |                                            |                                                              |                                                              |
| xxx      | NSDI24    | Serverless                                 | [Autothrottle: A Practical Bi-Level Approach to Resource Management for SLO-Targeted Microservices](https://www.usenix.org/conference/nsdi24/presentation/wang-zibo)<br />***Outstanding Paper\*** | Learning-based resource management                           |
| xxx      | NSDI24    | Serverless                                 | [Jolteon: Unleashing the Promise of Serverless for Serverless Workflows](https://www.usenix.org/conference/nsdi24/presentation/zhang-zili-jolteon) | Auto resource allocation algorithm                           |
| xxx      | NSDI24    | Serverless                                 | [Can't Be Late: Optimizing Spot Instance Savings under Deadlines](https://www.usenix.org/conference/nsdi24/presentation/wu-zhanghao) <br />***Outstanding Paper\*** | Better scheduling policy for spot instances to meet deadlines with extra on-demand instances |
| xxx      | NSDI24    | Serverless                                 | [Towards Intelligent Automobile Cockpit via A New Container Architecture](https://www.usenix.org/conference/nsdi24/presentation/jiang-lin) | Better Android container for cars                            |
| xxx      | NSDI24    | Serverless                                 | [MuCache: A General Framework for Caching in Microservice Graphs](https://www.usenix.org/conference/nsdi24/presentation/zhang-haoran) | for intelligent cockpitsShared cache to speed up cross-function invocation |
|          |           |                                            |                                                              |                                                              |
