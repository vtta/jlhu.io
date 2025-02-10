+++
+++
尽管之前偶然测得我的设计能到90%的DRAM ratio. 但是后来无法复现, 且波动巨大. 怀疑是PEBS引入的不稳定性, 所以想看看到底PEBS到底怎样才能带来最好的结果.

之前实验发现使用load+store事件并不能有效的发现cold folio, DRAM heap也非常小. 后来改为保留一个全局事件, 并使用一个独立事件抓取尽量多的DRAM访存信息, 希望这样可以尽可能多的发现DRAM cold folio.

这里改变了两个事件的sample period来看看这个设计有效与否. 同时每个setting同时运行5个VM, 防止出现偶然事件.

<https://web.pc49058.tunnel.jlhu.io/data/projects/workspace/archive/2024-09-10T23:44:28.608346+08:00-sample_frequency-all/report-period.html>

从目前的结果看来基本上还是没法发现cold folio. Hotset DRAM ratio最高也才不到三分之一, 即便DRAM heap中已经有了全部的DRAM folio (≈250k).

考虑将DRAM的hotset识别改为排除式, 即维护一个所有DRAM的set. 从PEBS sample中观测到的folio全部列为hot. 其余全部尝试做demotion. 直到PMEM heap中不在有candidate.

这里的假设是: 在sample frequency较低时, 采样到的地址大概率就是hot的. 先用perf record做个实验, 记录不同frequency下采样地址属于hot set的概率.

需要的改动: ~~gups需要输出hotset的区间地址.~~ 已经实现了, 输出格式: `memory 0x7fc548c000d0 length 13950255104`

<https://web.pc49058.tunnel.jlhu.io/data/projects/workspace/archive/2024-09-11T11:04:15.586819+08:00-event_selection/0/gups/event-selection.html>

从目前看sample frequency和P(hotset)关系不大, 反而事件的选取更能影响P(hotset). 并且确实sample中存在hot的bias, 比如选取L3Miss事件时约87%的sample都是在访存hotset.

所以现在只要保重余下的访存能够touch到足够多的DRAM区域, 以提供demotion的candidate. 这样我们就可以单纯只靠这一个event来即识别hotset有找到demotion candidate.

那这个事件到底该是l3miss还是loadlat? 从图中看他俩最终P(hotset)差距不大, 但是sample的量却差了好几个数量级, 我们当然是希望用更少的sample来识别出hotset, 即开销更低.

所以接下来做一个实验, 把之前的store event换成ldlat, 然后local load event先不动, 理想情况下在这个额外事件sample frequency很低的时候仍然能达到较高的hotset ratio就说明之用一个ldlat事件是可行的.

<https://web.pc49058.tunnel.jlhu.io/data/projects/workspace/archive/2024-09-11T22:30:56.439396+08:00-sample_frequency-ldlat-all/report-period.html>

实验效果很差, 不管ldlat和dram event怎么变化, hotset ratio都低于1%. 理论上87%sample属于hotset, 那么仅看页地址的sample计数怎么说也能发现大部份hotset. 目前怀疑是decaying引起的. 尝试关闭decay, 回退到最原始的sketch只看总计数.
