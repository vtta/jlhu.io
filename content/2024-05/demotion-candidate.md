+++
title = "Demotion Candidate"
date = "2024-05-01"

+++



~~问题: PEBS+SDH的组合能较好识别热数据, 但是冷数据怎么识别?
我们改了migration的设计, 采用exchange的方式. exchange需要在开始之前就准备好promotion和demotion的页, 即热页和冷页. 
热页比较好处理, 直接将PEBS的sample通过SDH过滤即可找出最~~



问题: SDH只能统计访问频率, 并不能区分冷热. 

我们最初的SDH设计也有这个问题, 不过当时是采用了aux heap结构来保存最hot的页. 



placement的核心有两种角度看:

1. 是将FMEM的冷页和SMEM的热页做交换;
2. 将全局的热页放在FMEM中;

第一种是稳态的维护, 而第二种算是最终目的. 

这俩种也对应了SDH的两种处理方式:

1. FMEM/SMEM分别维护SDH: 这种情况下FMEM的aux heap用来保存最cold的页; SMEM的aux heap用来保存最hot的页. 
   问题来了, 保存最cold的页实际上无法实现, 因为aux heap只会记录已经见过(PEBS sample到)的页. 而最cold的页被sample到的概率太小.
2. 全局维护一个SDH: 这种情况aux heap的大小=FMEM的页数, 按照8G算大概为2 million.  可以简单测一下2M大的heap的平均插入时间. 目前测得2\~3us. 
   这里的问题就是如何知道aux heap中的页全部都处在FMEM中? 可以额外增加两个set, 即demotion set和promotion set. 每次看到一个sample时, 如果发现其应被插入aux heap并且是SMEM页, 则同时将其加入promotion set; 类似如果发现其不属于aux heap并且是FMEM页, 则同时将其加入demotion set.
   这里另一个问题还是存在, demotion set可能非常难以被填充. 因为sample到SMEM页的概率太小了. 
   由PEBS驱动的被动填充目前看来是不行的. 还得是actively checking最好. 目前linux/TPP的demotion step也是actively checking. 不过他们check的是access bit, 我们可以直接check SDH记录的访问次数.





Decaying与aux heap的维护

decaying会将conflict的counter值减一, 但是我们并不知道被conflict的key, 即地址是什么. 所以我们无法对aux heap做及时的更新. 这也意味着aux heap和SDH是不同步的, 这样维护起来非常麻烦. 现在还得考虑一些新的decaying方法, 或者aux heap的新设计.

已准备好gups-hotset的sample trace: `gups-hotset-4c16g-0.4-nothing-2023.perf.data.pfn.csv` (总共5M+ sample)可以用来模拟decaying的设计

先模拟确定好再实现再做实验.

### What's the fundamental purpose of a hotness management data structure?

帮我们确定哪些页是热(应被放在FMEM)以及冷(应被放在SMEM).

(hot-tracking vs cold-tracking?)

分类:

- Distribution-based (Placement-oriented) 即通过全局的考量来放置, 通常暗示了需要估算每个页的访问频率.
  - Sorting (vTMM): 最简单最暴力的方法. 按照访问频率排序完后, 将靠前的页放在FMEM, 靠后的页放在SMEM.
  - Histogram (Memtis): 算是一种优化, 并不需要精确到每个页的顺序. 按bucket排序后, 靠前放FMEM, 靠后放SMEM.
- Eviction-oriented
  - TPP/Linux's LRU: LRU本身是为cache eviction设计的, 即找出应被剔除的页来腾出空间; 由于对热页的区分度不够, 所以不适合作为唯一的hotness management solution.
  - Autotiering's LFU: 为每个页维护八次hint fault中是否发现的历史(bitmap). 如果八次都没访问则会被demote. 这个方法也是cold management solution. (八次都没置位说明很久都没有访问, 但八次置位不能代表最近就有被经常访问) 但因为每个页所对应的八次历史并不是对齐的(每次hint fault scan只会扫描256MiB的页). 即便某个页的bitmap都是置位的, 其也有可能在最近很多次scan中都没被发现. (因为没有扫描到他, 没有unmap, 没被访问也不会被发现). 
    (这里也可以看到hint fault和A-bit的一个区别: hint fault中即便一个页被访问了也可能没被发现, 但是一个页被访问了A-bit一定会被置位).
    (Autotiering没有promotion时的hot selection mechanism, 只有demotion时的cold selection mechanism; 并且完全是hint fault based)

(Misplaced-tracking vs goal-tracking?)



SDH的问题

SDH的核心idea: distribution-based方法实际上只需要维护访问频率间的偏序关系, 即要放到FMEM的页访问频率高于要放到SMEM的页.

sketch本身只能记录访问频率, aux heap才是主要结构. 采用min-heap正好就是为了这个偏序关系. heap-min就是FMEM-SMEM的分界访问频率. heap中的页都大于这个分界线, 所以都应被放到FMEM. 采用min-heap也不需要memtis那种和deeply nested记录方式. 也可以避免log形式histogram的区分度问题 (都挤在某几个bucket里, 这个存在么???).

但是以上只是相当于goal-tracking, 只是理想情况. 我们还得需要知道实际情况, 以及如何根据实际情况做出调整来达到理想情况.

- 最简单的调整方式就是定期扫描所有的页, 看看heap中记录的值是否和aux heap中的值一致.
- 也可以由别的方式触发: 
  - 可以在heap偏大到某个阈值时做scan/calibrate. 因为decaying的存在, 可能heap中的元素本不应该存在其中, 所以heap有偏大的趋势.
- misplaced-tracking可以同时在扫描的过程中进行, 如果一个页在heap中, 但是不属于FMEM则可以加到promotion候选, 如果不在heap但是在FMEM则可以加到demotion候选. 





- 迁移完成后, sketch中的计数会和aux heap不吻合. 因为sketch没法直接设置一个值, 我们没法交换promotion/demotion两个页的访问计数
- 这里得设置一个cooldown, 每个页migrate之后过多长时间才能允许第二次被migrate, 如何实现? cooldown set?



- 整理目前所需要的全部数据结构
- 
