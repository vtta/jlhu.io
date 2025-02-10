+++
+++
### 一句话概括

现有OS级的TM支持已经很普遍, 但是虚拟化环境对TM的支持还比较欠缺. 对此我们提出一套针对虚拟化环境TM的解决方案. 这套方案包括以下几部分: 1. 如何将TM提供给guest; 2. 如何让guest利用好TM. 这套方案能让性能提升xx倍.

### 细说

"现有OS级的TM支持"以及"虚拟化环境"主要想讨论的是本文的对比对象: OS级主要包括Memtis/TPP/Nomad; 次要包括Autotiering/Nimble/Thermostat; VMM级包括Memstrata/vTMM/RAMinate/HeteroOS/HeteroVisor. 主要借鉴对象还包括DaOS.

"普遍"!=好. 这里只是合理化我们认为应该在OS级做TM的管理. 但是也正是因为这个选择, 我们希望OS的TM管理能力要做的更好. 突出"普遍"的写法可以借鉴[SOSP19]Recipe intro para 3.

| 现有方案    | 贡献                                                         | 防守                                                         | 攻击                                                         |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Memtis      | 1)改进了热区检测(distribution-based); <br />2)发现了巨页内部的不均衡问题; | 1) 我们不仅考虑distribution还有locality;<br />2)巨页在VM环境中支持并不好[^1], 暂时不考虑. | 页访存信息存在页表中且无硬件支持, 维护成本过大               |
| TPP         | 提出了主动页迁移(proactive demotion)[^2]                     | 和我们不冲突, 我们保留了wmark机制, 保留足够FMEM headroom.    | 依赖于hint fault做profiling. 开销过大.                       |
| Nomad       | 提出了shallow page copy                                      | ==(待验证)对容量的影响如何?==                                | 对于非只读访问容易陷入dirty-retry loop                       |
|             |                                                              |                                                              |                                                              |
| AutoNUMA    | 提出了Linux的内存迁移支持                                    |                                                              | 只有最基本的迁移能力. 在profiling/migration方面均不是最优.   |
| Autotiering | 补充了多层级间的migration path.                              | 虚拟机内存不会太大, 我们暂时只考虑FMEM/SMEM两层情况.         | 只是针对AutoNUMA                                             |
| Nimble      |                                                              |                                                              |                                                              |
|             |                                                              |                                                              |                                                              |
| Memstrata   | 1)提出FlatMemMode;<br />2)改进了VM间的性能隔离               | 1) FMM等价于SMEM, 于我们的模型不冲突;<br />2) 以页为单位的管理不会出现竞争; | 1) 特殊硬件, 通用性受限;<br />2) 无法保证FMEM的有效利用      |
| vTMM        | 1) 改进sampling (PML加速pgtbl scanning);<br />2)             |                                                              | 1) 开销, 且引入isolation的问题以及与guest自己的管理冲突;     |
| RAMinate    |                                                              |                                                              |                                                              |
| HeteroOS    |                                                              |                                                              |                                                              |
| HeteroVisor |                                                              |                                                              |                                                              |
|             |                                                              |                                                              |                                                              |
| DaOS        | 提出了DAMON框架(space-based sampling)                        |                                                              | 目前仅支持PTE-A bit作为数据来源, 不支持PEBS. <br />仅支持用户态管理 |

[^1]: [EuroSys23]Gemini中提出了巨页在hypervisor以及guest OS之间的proper alignment问题.
[^2]: 利用了[ASPLOS23]TPP sec 3.6中''新≈热"的观察.

[AutoNUMA细节](../2023-08/kernel-memory-tiering-status.md)   [TPP细节](../2024-06/tpp-migration-details.md)

#### 大背景: 云环境

为什么需要TM? 内存容量的发展速度落后于CPU core增长的速度. 需要新的介质或者interconnect来补足.

为什么云需要TM? 为了保持现有VM的mem/core比例. 扩容, 降本.

云的定义? 一些虚拟机的集合.

虚拟机是典型/基本单元么? 云计算产品一般包括EC2/bare-metal/serverless. EC2显然就是VM. BM实际上是运行在硬件VMM上的虚拟机. 而serverless底层运行环境是managed VM.

我们假设VM暂时只有两层, 有几到几十个core以及\~10G到\~100G内存. (这里的数据支持? 看看memstrata sec2.1)

TM在云环境下的构架选择?

### 文章大纲 v0.0

#### Abstract 250

#### Intro 1250

1. Motivation
   1. TM可以解决云面临的内存scaling瓶颈 (图?)
2. Limitation 1
   1. OS已经有普遍的TM支持, 但是虚拟化环境下缺少成熟方案.
   2. 现有虚拟化解决方案要么不支持placement管理要么必须费劲抓取guest信息.
   3. 一套OS TM系统到虚拟化环境的转化方案
3. Limitation 2
   1. 转化OS TM后sampling/classification开销仍不理想
   2. 一套针对OS内低开销TM管理方案
4. Contribution
   1. Hetero balloon:
   2. Budget constrained hotness detection
   3. Range/locality based FMEM management
5. 结果很好

#### Background and Motivation 2000W 2F V0.0

1. 云环境选择使用TM降本扩容[^b1]
   ==移到1.1==
   1. 虚拟机是云架构的基础
   2. 内存发展落后于core增长, 限制了云scalability
   3. TM的定义[^b2]
   4. 使用TM扩容需要降低SMEM的开销
   5. 利用data placement是核心方法
2. 现有OS对TM的支持很普遍
   1. AutoNUMA早在十年前就引入了最基本的TM支持
      1. 以NUMA形式管理TM, promote on hint fault
      2. ~~问题1: sampling (hint fault) 以及classification (split-LRU traversing) 都开销巨大~~
      3. 问题2: 仅在访问时迁移
   2. Nimble/Autotiering/TPP/Nomad后续继续改进
      1. Nimble支持巨页迁移/Autotiering支持多层级且介质优先的迁移/TPP支持进程调度时按内存压力触发迁移/Nomad支持异步迁移
      2. ~~均没有摆脱sampling以及classification的巨大开销~~
   3. Memtis尝试变革sampling
      1. 引入硬件加速的sampling方法
      2. ~~classification的开销仍没有解决~~
3. 云环境TM的架构应提供接口让gOS管理TM
   1. ~~hotness tracking无法全部在在VMM中完成 [^b3]~~
      1. ~~Memstrata就压根不支持hotness管理~~
      2. ~~vTMM热度识别依赖于扫描guest页表~~
      3. ~~HeteroOS guest向VMM指定扫描地址~~
   2. ~~需要兼容guest-aware的NUMA形式的OS支持~~
      1. ~~现有OS-based都是基于NUMA形式~~
      2. ~~Memstrata/vTMM等unified address space无法让gOS有所作为~~
   3. ~~TM形式需要动态以及细粒度~~
      1. ~~overcommit以及sharing要求动态调整支持~~
      1. ~~hotplug/virito-mem需要连续物理内存~~
   4. 虚拟化引入的GVA/GPA的地址转换隐藏了应用热度信息
      1. locality一般存在于GVA空间, 但hypervisor只可以见GPA
      2. 页表可以将GVA转化成GPA
      3. 侵入性读取guest内存进行翻译
   5. HeteroOS采用以上方法
      1. guest提供GVA区间
      2. hypervisor只能管理GPA
      3. 使用页表转换
      4. 读guest内存算security breach
   6. vTMM也逃不过
4. 云环境TM需要最小的CPU开销
   1. CPU利用率直接影响到成本和利润
   1. 现有hypervisor开销巨大
      1. 需要额外的地址转换来管理hotness
      1. para-virt选择通过page fault来mirror pgtbl
      1. hw-virt选择通过监控pgtbl地址
   1. 现有Linux-derived的设计sampling及classification均开销巨大 (加图: cycle per sample)
      1. sampling
         1. pgtbl scan
      1. classification
         1. LRU rotate
      1. migration
         1. hint fault
   1. IS准确且开销低且通用且隐私好
   1. Memtis的Cold detection很差
5. IS的guest态支持

[^b1]: Azure: Pond/Memstrata; Amazon: DaOS; Meta: TPP; Google: TMTS;
[^b2]: TM的定义划分时应将FlatMemMode化为SMEM.
[^b3]: 例如Memstrata中的Figure 1. VMM看到的GPA空间不存在locality.

### 文章大纲 v0.1

#### Intro

1. 云环境选择使用TM降本扩容[^b1]
   1. 虚拟机是云架构的基础
   2. 内存发展落后于core增长, 限制了云scalability
   3. TM的定义[^b2]
   4. 使用TM扩容需要降低SMEM的开销
   5. 利用data placement是核心方法
2.

#### Background and Motivation 2000W 2F

1. hypervisor假设OS没有TM管理能力
   1. HeteroVisor
      1. 以CPU的p-state作为灵感, 将混合内存以e-state形式包装给guest.
      2. guest通过调节的e-state数值向hypervisor要求提高内存的性能.
      3. hypervisor会完全透明的将guest的物理页更多的映射到FMEM.

   2. HeteroOS
      1. 尽管引入了guest OS awareness.
      2. 还是认为guest OS缺少硬件权限做hotness tracking.
      3. 于是将TM管理交给hypervisor.

   3. vTMM 及 Memstrata
      1. 均采用hypervisor only的TM管理.
      2. vTMM由hypervisor全权负责sampling/classification/migration.
      3. Memstrata则完全放弃了软件TM的placement管理, hyperisor仅管理VM分配到的FMEM/SMEM数量.
2. 现有OS对TM支持很普遍 => 设计hballoon将OS的TM管理引入虚拟化环境
   1. 尽管AutoNUMA很早就支持以NUMA形式暴露TM.
      1. 以NUMA形式管理TM, promote on hint fault
      2. ~~问题1: sampling (hint fault) 以及classification (split-LRU traversing) 都开销巨大~~
      3. 问题2: 仅在访问时迁移

   2. 过渡段
      1. AutoNUMA落后的data placement管理让hypervisor TM走上了hypervisor only的道路
      2. 但最近五年OS TM的爆发使得我们应重新审视这条道路.

   3. Nimble/Autotiering/TPP/Nomad后续继续改进
      1. Nimble支持巨页迁移/Autotiering支持多层级且介质优先的迁移/TPP支持进程调度时按内存压力触发迁移/Nomad支持异步迁移
      2. ~~均没有摆脱sampling以及classification的巨大开销~~
   4. Memtis尝试变革sampling
      1. 引入硬件加速的sampling方法
      2. ~~classification的开销仍没有解决~~
3. 云TM需要最小CPU开销 => 设计HPlacement优化OS的TM管理
   1. CPU利用率直接影响到成本和利润
   2. Hypervisor-based开销和安全性成问题.
      1. 需要额外的地址转换来管理hotness
      2. para-virt选择通过page fault来mirror pgtbl
      3. hw-virt选择通过监控pgtbl地址
   3. 现有Linux-derived的设计sampling及classification均开销巨大 (加图: cycle per sample)
      1. sampling
         1. pgtbl scan
      2. classification
         1. LRU rotate
      3. migration
         1. hint fault
   4. IS准确且开销低且通用且隐私好
   5. Memtis的Cold detection很差
4. guest IS: 硬件支持且有privacy保证
   1. IS如何工作
   2. Intel pebs guest的支持
      1. isolation很好

#### Design 3000W 3F

1. Principles
   1. Provide a generic way to expose and resize different memory tiers
   2.

2. Hetero-balloon 作为接口
   1. Per-node sub-b/alloon
   1. 细粒度动态内存的分配与回收
3. overhead constrained gOS hotness management
   1. 事件触发架构
   1. 低开销的sampling
   1. 动态低开销的segment tree classification
   1. page exchange

### 文章大纲 v0.2

#### Design 3000W 3F

1. Overview
   1. 设计HBalloon将OS的TM管理引入虚拟化环境
   2. 设计HPlacement优化OS的TM管理

2. HBalloon: 转化OS TM到云TM
   1. 采用NUMA node的形式暴露TM
   1. 细粒度动态内存的分配与回收
3. ~~overhead constrained gOS hotness management~~
   1. ~~事件触发架构~~
   1. ~~低开销的sampling~~
   1. ~~动态低开销的segment tree classification~~
   1. ~~page exchange~~
4. HPlacement
   1. Range-based management
      1. Locality

#### Implementation 400W

1. Base system and version
2. LoC stats
3. How PMEM is used

#### Evaluation 3000W 10F

1. 主要目标
2. 实验环境
3. HB+gOS的可行性+兼容性实验
   1. HB 对比 VB (X=vmnum Y=gups C=balloon)
   2. HB的resize性能 (X=time Y=size C=balloon)
4. 与多gOS对比: HA能更高效的利用FMEM
   1. 多VM下多scalability (X=vmnum Y=gups C=os)
   2. (hotness detection)时序图对比反应速度 (X=time Y=gups C=os)
   3. 开销breakdown  (X=stage Y=cycle C=os)
   4. paired migration对比exchange (X=method Y=throughput)
5. HB+HA的通用性
   1. realworld workload (X=workload Y=speedup C=os)
6. CXL version

#### Discussion 1000W

*这里应该放第二章没有串起来但是又比较重要的文章.*

1. Hardware tiering/caching:
   1. PMEM Memory Mode
   2. Flat Memory Mode: 仍然属于SMEM的范畴.

2. hotplug and virtio-mem [^s1]
   1. hotplug最小128MiB的粗粒度
   2. 均需要空出连续的物理内存

3. VM内存的动态平衡.
4. PML logging
5. guest instruction sampling
6. Para-virtualization (HeteroVisor) and hardware-virtualization
7. Colloid
   1. placement storm
      1. 因为不同VM share同一套uncore CHA, 一个VM的placement决策会由CHA传导到所有VM, 导致placement storm以及bouncing

   2. isolation/security
      1. hypevisor可以虚拟化CPU core以及per core PMU, 但是无法虚拟化shared uncore CHA hardware
      2. 共享uncore CHA可能引发side channel attack

   3. 开销
      1. Colloid可能(Colloid-TPP)需要一个单独的core做spin polling.

[^s1]: [VEE21]virtio-mem

#### Conclusion 100W

#### References

主要: Memtis/TPP/Nomad; Autotiering/Nimble/Thermostat; Memstrata/vTMM/RAMinate/HeteroOS/HeteroVisor; DaOS.

次要: Pond

### 文章形式

| Paper             | Page   | Abstract | Intro    | Background  | Design   | Implementation | Evaluation | Discussion | Conclusion | Total     |
| ----------------- | ------ | -------- | -------- | ----------- | -------- | -------------- | ---------- | ---------- | ---------- | --------- |
| ChatGPT           | 12     | 250      | 1250     | 1000        | 2250     | 1250           | 2250       | 1000       | 400        | ~10000    |
| [SOSP23]Memtis    | 14     | 192      | 1065     | 2096        | 3629     | 229            | 3531       | 821        | 77         | 11640     |
| [OSDI24]Memstrata | 14     | 242      | 1053     | 1585 + 2674 | 2518     | 286            | 2497       | 753        | 81         | 11689     |
| [EuroSys23]vTMM   | 13     | 268      | 1218     | 1409        | 2370     | 132            | 5160       | 0          | 90         | 10647     |
| **Target**        | **12** | **250**  | **1250** | **2000**    | **3000** | **400**        | **3000**   | **1000**   | **100**    | **11000** |

#### Motivation的figure如何处理?

- Memstrata:
  - Figure 1: 展示了guest physical address space上的spatial locality随着时间推移逐渐消失. 结论是在虚拟化环境下hotness profiling没法利用spatial locality降低开销
  - Figure 2: 展示了硬件Tiering (PM MemMode) 好过软件Tiering. 目的是为Flat Mem Mode的提出做铺垫.
- Memtis
  - Figure 1: 展示了不同参数的DAMON的hotness detection结果. 结论是页表扫描要么开销太高要么精度不够. 目的是为提出使用PEBS做hotness detection做铺垫.
  - Figure 2: 展示了现有PEBS-based solution不能较好的match hot size和DRAM size.

Motivation章节的精髓就是展现出SOTA不好但是我们好的所有点, 但又不明讲我们的设计, 吊足读者胃口.

### Misc findings

#### Linux's usage of PTE.A bits

The first thing unusual is that Linux refer to x86-64's PTE.A bit as "young" bit, the corresponding test function is `pte_young()`. The PTE.A bits scanning code calls `ptep_test_and_clear_young()` in the end, so that the accesses between two rounds of scanning can be discovered.

The main entrance for LRU scanning is `shrink_node()`. It can be called by kswapd in background when free memory dips below the low watermark. Or it will be called by the allocator directly when free memory dips below the min watermark, which signifies a severe memory shortage.

`shrink_node()` will eventually walk every anon/file in/active lru located in each `memcg`. Linux's LRU is a split-LRU, consisting of a pair of active and inactive lists. The scanning logic will first rotate the active list, and move any page, which has no PTE.A references by any page table via a call to `page_fererenced()`, to the inactive list.

For the inactive list, things are a bit more complicated. The scanning logic also rotates the list. Any page without any PTE.A references will be reclaimed. If they have PTE.A references, they are kept on the inactive list for another round. If a page is found with PTE.A references two times in a row, they are moved back to the active list. In the source code, this "two time in a row" logic is implemented using the `PG_referenced` bit of `struct page`, see `page_check_references()` for the details.

The "reclaim" means two possible outcomes. The first one is demotion to a lower node, the second one is swapping/discarding for anon/file pages.

因为LRU scanning是消极触发的, 如果我们想体现他的开销, 可以想办法让内存水位维持处于一个较低值. 这样kswapd就会一直触发. 对于TPP来说, 因为他有主动demote机制, 只要调高demote watermark就比较容易展现. 对于TPP之前的设计, 只能通过修改promotion来间接让可用内存降低从而触发kswapd.

#### TM mgmt overhead breakdown

对于Linux-based, 因为其没有一个中心的TM管理者, 想要做一个开销统计, 首先得知道各个切入点. 目前可以采用两种维度切入. 第一个是promotion/demotion的角度. 第二个则是sampling/identification/migration的角度.

- pro/demotion
  - promotion的切入点是hint fault. 如何触发hint fault; hint fault又如何触发promotion都可以作为参考. 但目前有一个问题, hint fault本身的开销没有一个好的方式去统计.
  - demotion的切入点则分为两个, 即为promotion分配空间时由于内存不足同步触发的direct reclaim. 以及异步触发的kswapd. 这两条路路径最后殊途同归到`shrink_node()`. 我们可以将其内发生的开销再进一步按sampling以及identification分. sampling可以包括rmap的walking. sampling之外的LRU scanning则可以划到identification. 总的来说涉及到pgtbl scanning的划为sampling, 涉及到lru的划为identification. 其他的划为migration.

##### Implementation

必须是侵入性修改. 基于bpftrace的funclatency.py虽然可以记录时间, 但是类似于scanned PTE.A bit count这里统计需要调用kernel函数, 没法通过bpftrace实现. 最终选择采用将每次调用的时延或计数记录到vmstat中来实现.
