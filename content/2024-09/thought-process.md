+++
+++
我们想提高虚拟化环境中内存的容量.

可选方案包括PMEM/CXL/RDMA/Swapping. 后两者由于性能是指数级别的区别的关系被排除.

PMEM/CXL是常数级别性能差别(~3x), 但是有指数级的容量优势: 和PMEM最大512G/DIMM以及CXL最大的256G/Module相比之下DDR4只有64G/DIMM.

同时相比有限的DIMM通道, CXL基于更多的PCIe通道, 另外PCIe也支持通过switching进一步提高扩展性.

那么PMEM/CXL容量不错, 该怎么利用起来呢?

首先, 我们称同时使用DRAM和PMEM/CXL的系统为tiered memory system (TM). DRAM称为fast memory (FMEM). PMEM/CXL称为slow memory (SMEM).

目前SMEM都支持以numa node的形式直接暴露为物理内存. 而物理内存只有host OS可见. hOS需要确定如何将内存分配给各个VM.

SOTA osdi24memstrata中是采取了hOS全权负责的模式. hOS检测哪些VM受慢内存影响较大, 然后给其分配更多的快内存.

memstrata这种方法没法使得分配出去的快内存映射到访问频繁的虚拟地址. 这种方法是对FMEM的浪费.

其他针对非虚拟化环境的TM设计可以让频繁访问的虚拟地址映射到FMEM, 从而增效降本. 包括Memtis/TPP等.

测试下SDS+IH的离线识别准确性? 这样可以为我们使用duty cycle做一点支持. 每隔一段时间牺牲下app运行时间, 全力把placement做到最好. 通过各个kthread的用时占比也可以看出确实是这块最耗时间.

做一个检测dram ratio的接口
