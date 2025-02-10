+++
+++
### Overhead breakdown

|           | sampling               | classification                           | migration                 |
| --------- | ---------------------- | ---------------------------------------- | ------------------------- |
| Ours      | overflow_handler       | policy                                   | migration                 |
| Memtis    | sampling_ns - ptext_ns | lru_rorate_ns + ptext_ns                 | demote_ns + promote_ns    |
| Nomad/TPP | ptea_scan_ns           | lru_rorate_ns - ptea_scan_ns - demote_ns | demote_ns + hint_fault_ns |

|        | sampling    | classification                           | migration                 | gups     | elapsed       |
| ------ | ----------- | ---------------------------------------- | ------------------------- | -------- | ------------- |
| Ours   | 867105721   | 20943846719                              | 6679106308                | 0.022998 | 32.396862731  |
| Memtis | 2716861-0   | 0 + 0                                    | 0 +2527774173             | 0.009398 | 79.281731536  |
| Nomad  | 80512427990 | 182161705169 - 80512427990 - 20700720145 | 20700720145 + 20329303765 | 0.005643 | 132.032089916 |
| TPP    | 7221328512  | 33551125323 - 7221328512 - 18339523530   | 18339523530 + 6296451214  | 0.021698 | 34.337351114  |

Memtis的主要问题: overshoot导致用光CPU quota, 无法可靠的连续采集访存样本.

Memtis的demote只会由page allocation触发. 在gups这种预分配内存的场景下demote_ns只会测得为0. 但是promotion则是以kthread形式间歇性执行. 其工作内容是扫描LRU list然后选择active anon作为promote对象. Promotion kthread只会在wmark允许的情况下执行promote.

### Memtis `cpu_quota`-gups curve

Memtis默认限制CPU开销在3%. 但是带来的结果就是gups非常低. 由上表中可以看到Memtis虽然相比其他设计开销低几个数量级, 但是gups只有不到TPP一半. 尝试增大`cpu_quota`看看gups表现.

| `cpu_quota` | Percentage | pebs_nr_sampled | Sampling                 | Classification           | Migration              | GUPS     | Elapsed      |
| ----------- | ---------- | --------------- | ------------------------ | ------------------------ | ---------------------- | -------- | ------------ |
| 30          | 39         | 7490627         | 9505095388 - 1919558951  | 8151093480 + 1919558951  | 0 + 842995391          | 0.010041 | 74.204129867 |
| 4000        | 45         | 10976739        | 11237885472 - 1790838536 | 8284482676 + 1790838536  | 0 + 969185781          | 0.010514 | 70.865151682 |
| 2000        | 46         | 10972799        | 11518200073 - 1955810414 | 8069035352 + 1955810414  | 0 + 1181147287         |          |              |
| 1000        | 46         | 10974236        | 11427093430 - 1720877666 | 7650819040 + 1720877666  | 0 + 866463523          |          |              |
|             |            |                 |                          |                          |                        |          |              |
|             |            |                 |                          |                          |                        |          |              |
|             |            |                 |                          |                          |                        |          |              |
|             |            |                 |                          |                          |                        |          |              |
|             |            |                 | sampling_ns - ptext_ns   | lru_rorate_ns + ptext_ns | demote_ns + promote_ns |          |              |

动态改`/sys/kernel/mm/htmm/ksampled_soft_cpu_quota`

```shell
clear@clr ~ $ ls -lah /sys/kernel/mm/htmm/
total 0
drwxr-xr-x 2 root root    0 Nov  7 06:13 .
drwxr-xr-x 8 root root    0 Nov  7 06:06 ..
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_adaptation_period
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_cooling_period
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_cxl_mode
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_demotion_period_in_ms
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_gamma
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_inst_sample_period
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_mode
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_nowarm
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_promotion_period_in_ms
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_sample_period
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_skip_cooling
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_split_period
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_thres_cooling_alloc
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_thres_hot
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_thres_split
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 htmm_util_weight
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 ksampled_max_sample_ratio
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 ksampled_min_sample_ratio
-rw-r--r-- 1 root root 4.0K Nov  7 06:09 ksampled_soft_cpu_quota
clear@clr ~ $ cat /sys/kernel/mm/htmm/*
100000
2000000
CXL-emulated: enabled [disabled]
500
4
100007
NO MIG-0, [BASELINE-1], HUGEPAGE_OPT-2, HUGEPAGE_OPT_V2
0
500
199
[enabled] disabled
2
2621440
1
2
10
10
50
30
```

```shell
clear@clr ~ $ ls -lah /sys/fs/cgroup/memtis/
total 0
drwxr-xr-x  2 root root 0 Nov  7 07:52 .
dr-xr-xr-x 12 root root 0 Nov  7 07:47 ..
-r--r--r--  1 root root 0 Nov  7 07:47 cgroup.controllers
-r--r--r--  1 root root 0 Nov  7 07:47 cgroup.events
-rw-r--r--  1 root root 0 Nov  7 07:47 cgroup.freeze
--w-------  1 root root 0 Nov  7 07:47 cgroup.kill
-rw-r--r--  1 root root 0 Nov  7 07:47 cgroup.max.depth
-rw-r--r--  1 root root 0 Nov  7 07:47 cgroup.max.descendants
-rw-r--r--  1 root root 0 Nov  7 07:51 cgroup.procs
-r--r--r--  1 root root 0 Nov  7 07:47 cgroup.stat
-rw-r--r--  1 root root 0 Nov  7 07:47 cgroup.subtree_control
-rw-r--r--  1 root root 0 Nov  7 07:47 cgroup.threads
-rw-r--r--  1 root root 0 Nov  7 07:47 cgroup.type
-rw-r--r--  1 root root 0 Nov  7 07:47 cpu.idle
-rw-r--r--  1 root root 0 Nov  7 07:47 cpu.max
-rw-r--r--  1 root root 0 Nov  7 07:47 cpu.max.burst
-rw-r--r--  1 root root 0 Nov  7 07:47 cpu.pressure
-r--r--r--  1 root root 0 Nov  7 07:47 cpu.stat
-rw-r--r--  1 root root 0 Nov  7 07:47 cpu.weight
-rw-r--r--  1 root root 0 Nov  7 07:47 cpu.weight.nice
-rw-r--r--  1 root root 0 Nov  7 07:47 io.pressure
-r--r--r--  1 root root 0 Nov  7 07:47 memory.access_map
-r--r--r--  1 root root 0 Nov  7 07:47 memory.current
-r--r--r--  1 root root 0 Nov  7 07:47 memory.events
-r--r--r--  1 root root 0 Nov  7 07:47 memory.events.local
-rw-r--r--  1 root root 0 Nov  7 07:47 memory.high
-r--r--r--  1 root root 0 Nov  7 07:47 memory.hotness_stat
-rw-r--r--  1 root root 0 Nov  7 07:48 memory.htmm_enabled
-rw-r--r--  1 root root 0 Nov  7 07:47 memory.low
-rw-r--r--  1 root root 0 Nov  7 07:47 memory.max
-rw-r--r--  1 root root 0 Nov  7 07:47 memory.max_at_node1
-rw-r--r--  1 root root 0 Nov  7 07:47 memory.max_at_node2
-rw-r--r--  1 root root 0 Nov  7 07:47 memory.min
-r--r--r--  1 root root 0 Nov  7 07:47 memory.numa_stat
-rw-r--r--  1 root root 0 Nov  7 07:47 memory.oom.group
-rw-r--r--  1 root root 0 Nov  7 07:47 memory.pressure
-r--r--r--  1 root root 0 Nov  7 07:47 memory.stat
-r--r--r--  1 root root 0 Nov  7 07:47 memory.swap.current
-r--r--r--  1 root root 0 Nov  7 07:47 memory.swap.events
-rw-r--r--  1 root root 0 Nov  7 07:47 memory.swap.high
-rw-r--r--  1 root root 0 Nov  7 07:47 memory.swap.max
-r--r--r--  1 root root 0 Nov  7 07:47 pids.current
-r--r--r--  1 root root 0 Nov  7 07:47 pids.events
-rw-r--r--  1 root root 0 Nov  7 07:47 pids.max
```
