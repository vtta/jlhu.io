+++
title = "Kernel Memory Tiering Status"
date = "2023-08-24"
# [extra]
# add_toc = true
+++

<!-- # Kernel Memory Tiering Status -->

*(as of v6.5-rc5)*

## Introduction
The kernel's memory tiering support organizes NUMA nodes into tiers.
Cold pages in the upper tier will be demoted down to the lower tier.
Hot pages in the lower tier will be promoted up to the upper tier.
There could be multiple tiers in total,
and each neighboring tier has the above-mentioned behavior.

The kernel's memory tiering has been under development since 2019
in the [vishal/tiering](https://git.kernel.org/pub/scm/linux/kernel/git/vishal/tiering.git/) tree.
It was finally merged into the mainline v6.1 piece by piece starting from August 2022, mainly through the following patch series:
1. ["NUMA balancing: optimize memory placement for memory tiering system", v13](https://lore.kernel.org/linux-mm/20220221084529.1052339-1-ying.huang@intel.com/)
2. ["memory tiering: hot page selection", v4](https://lore.kernel.org/lkml/20220622083519.708236-1-ying.huang@intel.com/)
3. ["mm/demotion: Memory tiers and demotion", v15](https://lore.kernel.org/linux-mm/20220818131042.113280-1-aneesh.kumar@linux.ibm.com/)
4. Other commits authored by HUANG Ying on [GitHub](https://github.com/torvalds/linux/commits?author=yhuang-intel)
5. [mm/migrate: demote pages during reclaim](https://lore.kernel.org/all/20210304235958.ECFA81E5@viggo.jf.intel.com/)

The memory tiering is built on top of Linux's NUMA balancing solution AutoNUMA.
AutoNUMA's goal is to place memory close to the task accessing it or place the task close to the memory it is accessing.
AutoNUMA relies on NUMA hinting page faults to collect access samples.
The hotness identification algorithm is basically MRU.

AutoNUMA has lots of problems, but it gives MemTier the basic infrastructure to work on.
- AutoNUMA's MRU cannot capture "real" hot pages. (Addressed in patch 2)

    => MemTier migrates pages based on **time taken for a hint fault to occur** instead of
    AutoNUMA's **first page triggering a hint fault**.
    This is a kind of MFU policy, but it's a bit difficult to understand.
    For AutoNUMA, the hint fault might have found a page that was unmapped ten rounds ago.
    (One round is a NUMA scanning period, roughly 60s).
    However, this page has very little possibility to be a hot page.
    A real hot page could have taken only one round to trigger a hint fault.

- AutoNUMA does not have migration overhead control. (Addressed in patch 2)

    => MemTier migrates pages with a limited rate.
    This is because the fastest migration does not always mean the best application performance.
    One such indicator is workload responsiveness, i.e. observed latency.

- The original watermark mechanism leaves not enough free space on the upper tier to allow promotion.

    => MemTier introduces a new watermark called **promo watermark**.
    During migration, if the target node is nearly full, `kswapd` is woken to reclaim memory until the `promo` watermark.
    The `promo` watermark is set higher than the `high` watermark so that enough space could be released to facilitate promotion.
    The demotion is implemented in the memory reclamation path.
    So, during the triggering of reclaiming towards the `promo` watermark, the cold page will be demoted.
    The cold page is identified by Linux's LRU, i.e. active and inactive lists.



## Sample collection
Samples for promotion and demotion come from different mechanisms.
- For promotion, access samples are collected through NUMA hinting faults.

- For demotion, samples come from the access bit located in page table entries.

    They are collected during memory reclamation via page table scanning.
    Memory reclamation happens not often, they are mostly triggered when usable memory is low,
    i.e. below the `low` watermark.
    But MemTier added the proactive reclamation when promotion is needed.
    Memory reclamation will be triggered to free enough memory, i.e. up to the `promo` watermark, to facilitate promotion.


## Hotness identification
Hotness identification is done differently during promotion and demotion.
- For promotion, the strategy is MFU.
    MFU chooses those pages with hint fault latency smaller than the threshold, see [Intro](#introduction) for explanation.
    NUMA hinting page faults require switching between user and kernel space and invalidating TLBs, which are all very expensive.

- For demotion, the strategy is LRU.
    
    This LRU is active and inactive lists.
    All `struct page`s managed by them will be scanned each round to count all the access bits referencing them.
    Those reference bits are actually in the page table containing a page backed by the physical page managed by this `struct page`.
    All those page tables are found out through [`rmap`](https://lwn.net/Articles/75198/) following the structures shown [here](https://static.lwn.net/images/ns/anonvma2.png).
    This process requires walking the entire page table, which is also expected to be costly.


## Page migration
Linux page migration is now rate-limited as mentioned in [Intro](#introduction).




## Misc

By inspecting the commit log up until v6.5-rc5,
we have 12 commits involving the keyword "memory tiering".<details>
  <summary>commit IDs</summary>

```bash
git log --all -i --grep "memory tiering" | tee memtier.log
cat memtier.log | rg ^commit\  | cut -d\  -f2
```
```
c7cdf94e9cd7a03549e61b0f85949959191b8a10
6085bc95797caa55a68bc0f7dd73e8c33e91037f
27bc50fc90647bbf7b734c3fc306a5e61350da53
992bf77591cb7e696fcc59aa7e64d1200b673513
c959924b0dc53bf6252793f41480bc01b9792570
c6833e10008f976a173dd5abdf992e492cbc3bcf
33024536bafd9129f1d16ade0974671c648700ac
a1a3a2fc304df326ff67a1814364f640f2d5121c
c574bbe917036c8968b984c82c7b13194fe5ce98
e39bb6be9f2b39a6dbaeff484361de76021b175d
5c7b1aaf139dab5072311853bacc40fc3457d1f9
2dd57d3415f8623a5e9494c88978a202886041aa
```

</details>

After inspecting these commits, we found the patch series of interest linked in [Intro](#introduction).

<!-- Out of these commits, these are the most important ones: -->

<!-- | ID      | Info                                                         | -->
<!-- | ------- | ------------------------------------------------------------ | -->
<!-- | e39bb6b | Patch series "NUMA balancing: optimize memory placement for memory tiering system", v13 | -->



<!-- ### Patch series "NUMA balancing: optimize memory placement for memory tiering system", v13 -->

<!-- <details> -->
<!--     <summary>commit details</summary> -->

<!-- > commit e39bb6be9f2b39a6dbaeff484361de76021b175d -->
<!-- > -->
<!-- > Author: Huang Ying <ying.huang@intel.com> -->
<!-- > -->
<!-- > Date:   Tue Mar 22 14:46:20 2022 -0700 -->

<!-- >     NUMA Balancing: add page promotion counter -->
<!-- >      -->
<!-- >     Patch series "NUMA balancing: optimize memory placement for memory tiering system", v13 -->
<!-- >      -->
<!-- >     With the advent of various new memory types, some machines will have -->
<!-- >     multiple types of memory, e.g.  DRAM and PMEM (persistent memory).  The -->
<!-- >     memory subsystem of these machines can be called memory tiering system, -->
<!-- >     because the performance of the different types of memory are different. -->
<!-- >      -->
<!-- >     After commit c221c0b0308f ("device-dax: "Hotplug" persistent memory for -->
<!-- >     use like normal RAM"), the PMEM could be used as the cost-effective -->
<!-- >     volatile memory in separate NUMA nodes.  In a typical memory tiering -->
<!-- >     system, there are CPUs, DRAM and PMEM in each physical NUMA node.  The -->
<!-- >     CPUs and the DRAM will be put in one logical node, while the PMEM will -->
<!-- >     be put in another (faked) logical node. -->
<!-- >      -->
<!-- >     To optimize the system overall performance, the hot pages should be -->
<!-- >     placed in DRAM node.  To do that, we need to identify the hot pages in -->
<!-- >     the PMEM node and migrate them to DRAM node via NUMA migration. -->
<!-- >      -->
<!-- >     In the original NUMA balancing, there are already a set of existing -->
<!-- >     mechanisms to identify the pages recently accessed by the CPUs in a node -->
<!-- >     and migrate the pages to the node.  So we can reuse these mechanisms to -->
<!-- >     build the mechanisms to optimize the page placement in the memory -->
<!-- >     tiering system.  This is implemented in this patchset. -->
<!-- >      -->
<!-- >     At the other hand, the cold pages should be placed in PMEM node.  So, we -->
<!-- >     also need to identify the cold pages in the DRAM node and migrate them -->
<!-- >     to PMEM node. -->
<!-- >      -->
<!-- >     In commit 26aa2d199d6f ("mm/migrate: demote pages during reclaim"), a -->
<!-- >     mechanism to demote the cold DRAM pages to PMEM node under memory -->
<!-- >     pressure is implemented.  Based on that, the cold DRAM pages can be -->
<!-- >     demoted to PMEM node proactively to free some memory space on DRAM node -->
<!-- >     to accommodate the promoted hot PMEM pages.  This is implemented in this -->
<!-- >     patchset too. -->
<!-- >      -->
<!-- >     We have tested the solution with the pmbench memory accessing benchmark -->
<!-- >     with the 80:20 read/write ratio and the Gauss access address -->
<!-- >     distribution on a 2 socket Intel server with Optane DC Persistent Memory -->
<!-- >     Model.  The test results shows that the pmbench score can improve up to -->
<!-- >     95.9%. -->
<!-- >      -->
<!-- >     This patch (of 3): -->
<!-- >      -->
<!-- >     In a system with multiple memory types, e.g.  DRAM and PMEM, the CPU -->
<!-- >     and DRAM in one socket will be put in one NUMA node as before, while -->
<!-- >     the PMEM will be put in another NUMA node as described in the -->
<!-- >     description of the commit c221c0b0308f ("device-dax: "Hotplug" -->
<!-- >     persistent memory for use like normal RAM").  So, the NUMA balancing -->
<!-- >     mechanism will identify all PMEM accesses as remote access and try to -->
<!-- >     promote the PMEM pages to DRAM. -->
<!-- >      -->
<!-- >     To distinguish the number of the inter-type promoted pages from that of -->
<!-- >     the inter-socket migrated pages.  A new vmstat count is added.  The -->
<!-- >     counter is per-node (count in the target node).  So this can be used to -->
<!-- >     identify promotion imbalance among the NUMA nodes. -->
<!-- >      -->
<!-- >     Link: https://lkml.kernel.org/r/20220301085329.3210428-1-ying.huang@intel.com -->
<!-- >     Link: https://lkml.kernel.org/r/20220221084529.1052339-1-ying.huang@intel.com -->
<!-- >     Link: https://lkml.kernel.org/r/20220221084529.1052339-2-ying.huang@intel.com -->
<!-- >     Signed-off-by: "Huang, Ying" <ying.huang@intel.com> -->
<!-- >     Reviewed-by: Yang Shi <shy828301@gmail.com> -->
<!-- >     Tested-by: Baolin Wang <baolin.wang@linux.alibaba.com> -->
<!-- >     Reviewed-by: Baolin Wang <baolin.wang@linux.alibaba.com> -->
<!-- >     Acked-by: Johannes Weiner <hannes@cmpxchg.org> -->
<!-- >     Reviewed-by: Oscar Salvador <osalvador@suse.de> -->
<!-- >     Cc: Michal Hocko <mhocko@suse.com> -->
<!-- >     Cc: Rik van Riel <riel@surriel.com> -->
<!-- >     Cc: Mel Gorman <mgorman@techsingularity.net> -->
<!-- >     Cc: Peter Zijlstra <peterz@infradead.org> -->
<!-- >     Cc: Dave Hansen <dave.hansen@linux.intel.com> -->
<!-- >     Cc: Zi Yan <ziy@nvidia.com> -->
<!-- >     Cc: Wei Xu <weixugc@google.com> -->
<!-- >     Cc: Shakeel Butt <shakeelb@google.com> -->
<!-- >     Cc: zhongjiang-ali <zhongjiang-ali@linux.alibaba.com> -->
<!-- >     Cc: Feng Tang <feng.tang@intel.com> -->
<!-- >     Cc: Randy Dunlap <rdunlap@infradead.org> -->
<!-- >     Signed-off-by: Andrew Morton <akpm@linux-foundation.org> -->
<!-- >     Signed-off-by: Linus Torvalds <torvalds@linux-foundation.org> -->

<!-- </details> -->

<!--   -->
<!-- This patch series address the need of  -->
<!-- extending existing hotness identification and page migration mechanisms from AutoNUMA to address: -->
<!-- - Promotion of hot pages in PMEM node: -->
<!-- - Demotion of cold pages in DRAM node: 26aa2d199d6f ("mm/migrate: demote pages during reclaim") -->





