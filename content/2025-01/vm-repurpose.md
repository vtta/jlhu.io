+++
+++
[前面提到](serverless-paper-list)\[SOSP24\]TrEnv的核心贡献是开辟了一条解决cold-boot的新思路, 摒弃了古早的reboot和到SOTA的restore, 最终提出了repurpose. 目前看没有一个特别明显的切入点能做出一个更大的contribution. 考虑follow-up他们.

他们repurpose的是container, 但如今主流的serverless runtime均采用vm[^vmbased]. 其文章5.1.3中也讨论了将TrEnv放到VM的情况. 但是其没有涉及mm-template如何打破虚拟化到barrier.

目前naive的方法就是直接采用mm-template将整个firecracker进程打一个snapshot. 但是这样其实没有区分出user state, 也不存在说复用vm的可能, 虽然将snapshot存在了CXL可以加快startup但是tradeoff掉了memory saving.

那么想要将vm做成repurposable, 需要对kernel进行修改, 能够剥离user state.

一个基本的思路就是做一个类似virtio-mem能hot-plug的内存设备. “snapshot”时将user state剥离到这个device上. 恢复时将device attach到新的vm. 但是这种方案需要基于kernel的snapshot/restore支持. 目前这方面的研究还停留在上古时期[^CRIK], mainline中并无实现. 此文此方法的难度深度广度都很大, 作为一整个PhD课题才算比较合理.

那么问题来了, 如果这么搞为什么不在VM内部运行containerd? 然后由外部统一集中管理snapshot和恢复? 死局?

[^vmbased]: 工业界部署的多为VM-based方案, 包括阿里的\[ATC21\]FaaSNet, AWS的[ATC23]Lambda, Facebook的[SOSP23]XFaaS

[^CRIK]: 例如[BLCR](https://crd.lbl.gov/divisions/amcr/computer-science-amcr/class/research/past-projects/BLCR/checkpoint-restart-publications/), 其github停留在了八年前, 且还是基于linux 3.10.

---

退一步, 如果我们基于TrEnv的container-based solution做TM支持该怎么办?

目前一个比较直觉的做法是在snapshot的时候做好hotness classification. 因为TM系统中内存搬的再好不如不搬, 一开始就做好热区识别可以尽可能抹除后续的管理开销. 再加上serverless本来就有较短的lifetime, 持续online profiling + migration的模式不适用于serverless环境. 目前TrEnv同时只支持一种介质, 即要么CXL要么RDMA, 还没有TM支持. 使用TM可以同时有CXL的速度以及RDMA的容量.

但是用TM的motivation是什么? 主要的优势在哪里? 实验该如何设计? 对比对象有哪些?

设计:

1. 打snapshot前跑profiling
2. 收集每个function launch之后的hot信息做后台优化
3. 优化CoW, 提前copy一些hot页为RW

---

---

设备

目前看最新的RDMA网卡应该是ConnectX-8. 其速度为800G, 但是其价格在40k左右.

上一代的ConnectX-7目前价格比较能接受. 速度为400G, 价格在3-10k左右.
