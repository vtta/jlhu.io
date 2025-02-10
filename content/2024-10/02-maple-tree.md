+++
+++
Maple Tree就是一个BTree. 普通BTree经常被用作存储key-value mapping的结构. 这里的value按照kernel community的说法是一个singleton. 即可以看作是value值域内的某一个具体的值. 而Maple Tree针对的value则是range. 这么设计来源于Maple Tree的使命, 即加速vm area (一段地址空间)的管理维护.

这里和我们的需求也很契合, 我们想按区间的方式维护访存频率. 给定一个地址, 我们能很快的查找或更新其所在区间的总频率.

我们需要的最基本的接口:

- 初始化
  `mt_init()`
- 查找: 给定地址, 查找区间频率.
  `mt_find()`
- 更新: 给定地址, 更新区间频率.
  `mt_find()`这里简单的find就足够, 因为作为Key的地址区间没变.
- 分裂: 给定区间, 分裂成两个.
  `mtree_insert_range()`
- 合并: 给定区间, 遍历前后相邻区间, 然后合并.
  `mt_prev()/mt_next()`

目前这里都只使用了基本接口, 省去了管理树内部状态的复杂性.

#### 处理流程

1. 对于每一个采样到的虚拟地址, 首先找出VMA;
2. 确认该VMA是否是已经被我们管理/发现;
   1. 新发现/VMA被修改
      1. 下文讨论???
   2. 在管理
      1. 通过mtree找到对应区间
      2. 更新频率
      3. 检查是否到了split interval以及是否需要split/merge
      4. 如果和相邻区间的频率差低于merge阈值则merge
      5. 如果和相邻区间的频率差高于split阈值则split
      6. 同时更新iheap
3. 根据iheap对mtree的排序进行exchange/migration.
   1. 检测当前fmem/smem容量
   2. 找到合适的exchange/migration组合使得fmem利用率最高同时又不至于引发ping-pong

#### 如何处理VMA的更新?

1. 首先是如何发现VMA的更新

   目前我们只支持private anon, 情况比较简单. VMA的更新一般是和周围VMA合并或者拆分. 另外就是用户手动的调用`munmap/mprotect/madvise/mbind`等syscall. 我们目前只采用简单的workload可以避免这些情况.
   目前已知vma前40B包含了起止地址以及各种flags. 如果对此做一下checksum就能验证是否被修改.
