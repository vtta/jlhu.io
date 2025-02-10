+++
+++
## TPP migration实验总结

TPP的问题根源在于allocator和promotion的冲突导致很难发生promotion. [Allocator会默认将DRAM node用到low mark.](https://github.com/torvalds/linux/blob/a38297e3fb012ddfa7ce0321a7e5a8daeb1872b6/mm/page_alloc.c#L4543) 但是[promotion不会promote到处于high mark以下的node.](https://github.com/torvalds/linux/blob/a38297e3fb012ddfa7ce0321a7e5a8daeb1872b6/mm/migrate.c#L2485) 理论上DRAM node水位低于high mark时kswapd就会介入释放足够内存供promotion使用, 这里不管是[分配时发现过低](https://github.com/torvalds/linux/blob/a38297e3fb012ddfa7ce0321a7e5a8daeb1872b6/mm/page_alloc.c#L2919)或[promotion时发现过低](https://github.com/torvalds/linux/blob/a38297e3fb012ddfa7ce0321a7e5a8daeb1872b6/mm/migrate.c#L2533)都会唤醒kswapd. 但是实际上kswapd并不能够释放内存. 对于anonymous内存, 原因是我们观察到其存在active/inactive list ping-pong效应. 对于file内存, 在使用gups workload时其占比可以忽略不计.

对于inactive list, kswapd会统计所有扫描和回收的页, 分别记录到[`PGSCAN_ANON/PGSCAN_FILE`](https://github.com/torvalds/linux/blob/a38297e3fb012ddfa7ce0321a7e5a8daeb1872b6/mm/vmscan.c#L1919)和[`PGSTEAL_ANON/PGSTEAL_FILE`](https://github.com/torvalds/linux/blob/a38297e3fb012ddfa7ce0321a7e5a8daeb1872b6/mm/vmscan.c#L1936)这几个counter. 在gups workload中, 回收(包括了demote)的anonymous DRAM为0.

|      | PGSCAN    | PGSTEAL |
| ---- | --------- | ------- |
| ANON | 428474011 | 0       |
| FILE | 506637    | 118603  |

那这些anonymous页最后都去了哪里呢?

[对于active list, kswapd不会直接回收内存](https://github.com/torvalds/linux/blob/a38297e3fb012ddfa7ce0321a7e5a8daeb1872b6/mm/vmscan.c#L1998). 他只会扫描LRU并且重新检查扫过的页面到底是否还是active. 如果active则重新放回LRU; 如果属于inactive则执行deactivate操作, 即放到对应的inactive anon/file list. [这里扫过的总数会记录到`PGREFILL`](https://github.com/torvalds/linux/blob/a38297e3fb012ddfa7ce0321a7e5a8daeb1872b6/mm/vmscan.c#L2024), [deactivate的总数则会记录到`PGDEACTIVATE`](https://github.com/torvalds/linux/blob/a38297e3fb012ddfa7ce0321a7e5a8daeb1872b6/mm/vmscan.c#L2081).

| PGREFILL  | PGDEACTIVATE |
| --------- | ------------ |
| 387281191 | 387236411    |

[kswapd会分别先后扫描anon inactive, anon active, file inactive, file active list.](https://github.com/torvalds/linux/blob/a38297e3fb012ddfa7ce0321a7e5a8daeb1872b6/mm/vmscan.c#L5682) 这个特别的顺序就导致了ping-pong的触发. 在扫描inactive list时, 没有成功将anonymous页demote (PGSTEAL==0), 反而放入了active list. 后续扫描active list时又deactivate, 放回inactive list.

(假设)如果将扫描顺序调换, 先扫active再扫inactive, 则会出现active内存直接被demote的情况. 这是split-LRU只能做binary categorization的天然劣势.

## Migration Path

目前知道demotion的触发是因为内存压力过大/水位过低, 但promotion呢? 大概猜测promotion是寄生于numa balancing中, 访问下层NUMA node的memory时才会检查是否需要进行promotion.

### 代码

`should_numa_migrate_memory()`

以前的numa balancing会在page flags记录上次访问的CPU, 并根据CPU affinity来决定是否迁移. TPP的patch中加入了根据页是否时处于最高层级来决定是否migrate.

来接着看看具体的触发时机. 下面的代码中看出: 在触发NUMA hinting page fault时, 会检查该页的tiering, 并执行migration.
[相关代码](https://gist.github.com/vtta/d4de019b0849d662d472d28b5002e01d)

需要知道的信息:

- 页迁移的方向: promotion/demotion
  - 迁移的数据量
  - 迁移的吞吐情况
- 被什么触发?
- 内存压力情况
-

### Promotion

核心代码:

1. 核心迁移逻辑: `migrate_misplaced_page()`. v5.15中只发现在NUMA hinting fault handler中有两处调用, 分别为小页和大页的处理函数`do_numa_page()`/`do_huge_pmd_numa_page()`.

2. 触发hinting fault scan的逻辑`change_prot_numa()`. v5.15中只发现有两处调用: 分别为`task_numa_work`以及用于支持用户通过`mbind()/move_pages()`手动的页迁移. 其中`task_numa_work`才是我们关心的, 他是NUMA balancing的核心. `task_numa_work`由调度器在时钟中断中通过`task_tick_numa()`触发. 触发的频率受到`numa_scan_period`和NUMA balancing的总运行时间`se.sum_exec_runtime`的控制. 具体算法见[cbee9f8](https://github.com/torvalds/linux/commit/cbee9f88ec1b8dd6b58f25f54e4f52c82ed77690) [lkml](https://lore.kernel.org/all/20121025124834.467791319@chello.nl/)

3. NUMA scan的参数设定, 这些参数可以通过`debugfs`查看以及修改: `/sys/kernel/debug/sched/numa_balancing`

   1. 核心变量: `numa_scan_period`: 每个scan_period中扫描scan_size大的内存.

      ```c
      /*
       * Approximate time to scan a full NUMA task in ms. The task scan period is
       * calculated based on the tasks virtual memory size and
       * numa_balancing_scan_size.
       */
      unsigned int sysctl_numa_balancing_scan_period_min = 1000;
      unsigned int sysctl_numa_balancing_scan_period_max = 60000;
      
      /* Portion of address space to scan in MB */
      unsigned int sysctl_numa_balancing_scan_size = 256;
      
      /* Scan @scan_size MB every @scan_period after an initial @scan_delay in ms */
      unsigned int sysctl_numa_balancing_scan_delay = 1000;
      
      ```

   2. period还受到DRAM内存所占比例的影响, 在每次NUMA fault中的migration完成后, 会更新`numa_scan_period`. 如果大部分内存已经在DRAM则增加period, 减少未来的scanning. 尝试追踪`update_task_scan_period()`观察相关参数的变化.

      ```bash
      #!/usr/bin/env bpftrace
      kretfunc:update_task_scan_period { 
       printf("update_task_scan_period(p=%p, shared=%lu, private=%lu) pid=%d tgid=%d period=%u", 
       args.p, args.shared, args.private, args.p->pid, args.p->tgid, args.p->numa_scan_period)
      }
      ```

   3.

`numa_pte_updates` 记录了NUMA scan扫过的页数. `numa_hint_faults`则记录了hint fault发生的次数.

观察迁移成功与否

| Function                 | Invoke  | Active Seconds | Success | Scanned  | Comments |
| ------------------------ | ------- | -------------- | ------- | -------- | -------- |
| `migrate_misplaced_page` | 150598  | 220            | 0       |          |          |
| `kswapd_shrink_node`     | 1392    | 23             | 692646  | 16423520 | (nid==1) |
| `shrink_inactive_list`   | 2917471 | 291            | 0       | 92416425 | (lru=0)  |

#### Details

```bash
set -l log 2024-06-12T15:56:46.758221+08:00-force_deactivate/0/numa_scan.log
# migrate_misplaced_page 
## invole
cat $log | awk '$2 ~ /migrate_misplaced_page/ { print $1 }' | wc -l
## success

## active seconds
cat $log | awk '$2 ~ /migrate_misplaced_page/ { print $1 }' | cut -d: -f 1,2,3 | sort -u | wc -l

# kswapd_shrink_node 
## active seconds
cat $log | awk '$2 ~ /kswapd_shrink_node/ && $2 ~ /nid=1/ { print $1 }' | cut -d: -f 1,2,3 | sort -u | wc -l
## nr_scanned
cat $log | awk '$2 ~ /kswapd_shrink_node/ && $2 ~ /nid=1/ { print $5 }' | cut -d= -f 2 | xq -s add
## nr_reclaimed
cat $log | awk '$2 ~ /kswapd_shrink_node/ && $2 ~ /nid=1/ { print $6 }' | cut -d= -f 2 | xq -s add

set -l log 2024-06-12T20:58:53.759614+08:00-force_deactive-shrink_list_log/0/shrink_list.log
# shrink_inactive_list
## success
cat $log | sed 's/[(),]/ /g' | awk '$4 ~ /lru=0/  { print $6 }' | cut -d= -f2 | xq -s add
## scanned
cat $log | sed 's/[(),]/ /g' | awk '$4 ~ /lru=0/  { print $5 }' | cut -d= -f2 | xq -s
## active seconds
cat $log | sed 's/[(),]/ /g' | awk '$4 ~ /lru=0/  { print $5 }' | cut -d: -f 1,2,3 | sort -u | wc -l
```

### Useful Info

- [funcgraph-retval now supported on v6.5+](https://patchwork.kernel.org/project/linux-trace-kernel/patch/20230306113447.215527-1-pengdonglin@sangfor.com.cn/)

- <https://www.cnblogs.com/pengdonglin137/p/17880808.html>
