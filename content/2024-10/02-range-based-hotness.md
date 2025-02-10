+++
+++
目前看来完全依赖PEBS做hotness identification没办法把开销压制到一个合理的范围. 可以汲取DAMON的设计, 用一个树来表示地址空间(或某个VMA). 这样只用把sample frequency调到很低就可以找出最hot的region.

需要搞清楚的:

- [ ] Segment tree和interval tree各自的特点是什么? 对于不规整的range该怎么办?

- [ ] 依照什么规则来分裂一个range?
  分裂依据是与相邻区域的访存频率之差. 差别太小则merge. 差别太大则split. 频率差会动态调整, 最终目标是使得总区域数处于一个设定的区间.
  访存频率是通过主动采样得到. 即每隔一定时间, 在每个区域中随机选取一个页, 若其被访存(A-bit), 则区域计数增加.
  
  >#### Region Based Sampling
  >
  >... Thus, for each `sampling interval`, DAMON randomly picks one page in each region, waits for one `sampling interval`, checks whether the page is accessed meanwhile, and increases the access frequency counter of the region if so. ...
  >
  > #### Adaptive Regions Adjustment
  >
  >... For each `aggregation interval`, it compares the access frequencies (`nr_accesses`) of adjacent regions. ...
  >

- [ ] VMA被改变了怎么办?
  目前VMA支持split/merge, 一般发生在内存区域的大小/权限被修改时, 比如munmap/mprotect/madvise/mbind等. 我们可以每次对某个内存区域进行维护时检查其起始/终止地址以及权限来检测VMA的修改·

树的选择

- [ ] segment tree
  可以方便的记录和查找各个区域的访存次数, 但是没法记录merge/split的信息.
- [ ] interval tree?
- [ ] maple tree
  kernel中的VMA到底是怎么管理的?

