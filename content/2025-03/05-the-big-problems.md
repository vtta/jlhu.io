+++
+++

- CXL.mem的应用到底会有哪些?
  - 对于CXL 1.1, 虽然任何用到内存的都可以使用CXL.mem. 但CXL.mem本质还是在performance和price之间的一个tradeoff. 真正memory intensive的workload只会继续用DRAM不会选择CXL.mem. 真正想要低成本的workload也不会使用CXL.mem, 直接swapping就行了.
  - 至于说容量的提升, 至少在2.0引入switching之前是享受不到这个优势的. CXL.mem和CPU直连的内存都用的同样的DIMM, 只不过是之前CPU分给内存通道的引脚拿来分给了PCIe. 但是有了switching之后就完全不一样了, 一个机器的内存容量可以视为无上限. 这种情况是特别需要内存的应用所极其想看到的的.
    - 目前可以想象的包括内存数据库, 云和虚拟化, 超算, AI推理等等.
  - 目前CXL相关的paper只是让大家认识这个硬件的一些特性, 可是到底该用在哪里?
- CXL.mem在云中的形式?
  - 最显而易见的就是作为虚拟机的内存. 云诞生的意义就是提高资源的利用率. 十个人的十台机器平均利用率都只有10%, 云提供了一个更经济的共享解决方案. 在这种共享中, 其实大家对performance并不是特别的敏感, 那么这里也是CXL.mem特别适合的一个环境.
-
