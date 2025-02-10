+++
title = "Memtis Reproduce"
date = "2024-01-17"
# [extra]
# add_toc = true
+++


## No sample is collected by PEBS

### PEBS in Memtis
<details><summary>System physical memory range</summary>

```dmesg
[    0.119359] ACPI: SRAT: Node 1 PXM 1 [mem 0x00000000-0xbfffffff]
[    0.119363] ACPI: SRAT: Node 1 PXM 1 [mem 0x100000000-0x2bfffffff]
[    0.119364] ACPI: SRAT: Node 2 PXM 2 [mem 0x2c0000000-0x53fffffff]
[    0.119386] NUMA: Initialized distance table, cnt=3
[    0.119389] NUMA: Node 1 [mem 0x00000000-0xbfffffff] + [mem 0x100000000-0x2bfffffff] -> [mem 0x00000000-0x2bfffffff]
[    0.119400] NODE_DATA(1) allocated [mem 0x2bffd5000-0x2bfffffff]
[    0.119593] NODE_DATA(2) allocated [mem 0x53b7d4000-0x53b7fefff]
[    0.119866] Zone ranges:
[    0.119869]   DMA      [mem 0x0000000000001000-0x0000000000ffffff]
[    0.119875]   DMA32    [mem 0x0000000001000000-0x00000000ffffffff]
[    0.119877]   Normal   [mem 0x0000000100000000-0x000000053fffffff]
[    0.119880]   Device   empty
[    0.119883] Movable zone start for each node
[    0.119886] Early memory node ranges
[    0.119889]   node   1: [mem 0x0000000000001000-0x000000000009ffff]
[    0.119892]   node   1: [mem 0x0000000000100000-0x00000000bfffffff]
[    0.119893]   node   1: [mem 0x0000000100000000-0x00000002bfffffff]
[    0.119895]   node   2: [mem 0x00000002c0000000-0x000000053fffffff]
[    0.119896] Initmem setup node 1 [mem 0x0000000000001000-0x00000002bfffffff]
[    0.119903] Initmem setup node 2 [mem 0x00000002c0000000-0x000000053fffffff]
[    0.119991] On node 1, zone DMA: 1 pages in unavailable ranges
[    0.120249] On node 1, zone DMA: 96 pages in unavailable ranges
```
</details>

<details><summary>Perf samples of local DRAM access</summary>

```bash
clear@clr ~ $ sudo perf record --count 1007 --phys-data --data --weight -z -vv -e MEM_LOAD_L3_MISS_RETIRED.LOCAL_DRAM:Pu -- /data/gups-hotset 4 200000000 8581545984 8 1065353216 9
Using CPUID GenuineIntel-6-6A-5
Attempting to add event pmu 'cpu' with 'MEM_LOAD_L3_MISS_RETIRED.LOCAL_DRAM,' that may result in non-fatal errors
After aliases, add event pmu 'cpu' with 'event,period,umask,' that may result in non-fatal errors
MEM_LOAD_L3_MISS_RETIRED.LOCAL_DRAM -> cpu/event=0xd3,period=0x186a7,umask=0x1/
DEBUGINFOD_URLS=
Compression enabled, disabling build id collection at the end of the session.
nr_cblocks: 0
affinity: SYS
mmap flush: 1
comp level: 1
maps__set_modules_path_dir: cannot open /lib/modules/5.15.19-htmm+ dir
Problems setting modules path maps, continuing anyway...
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x1d3
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  precise_ip                       3
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 499  cpu 0  group_fd -1  flags 0x8
sys_perf_event_open failed, error -22
decreasing precise_ip by one (2)
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x1d3
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  precise_ip                       2
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 499  cpu 0  group_fd -1  flags 0x8
sys_perf_event_open failed, error -22
decreasing precise_ip by one (1)
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x1d3
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  precise_ip                       1
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 499  cpu 0  group_fd -1  flags 0x8
sys_perf_event_open failed, error -22
decreasing precise_ip by one (0)
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x1d3
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 499  cpu 0  group_fd -1  flags 0x8
sys_perf_event_open failed, error -22
switching off PERF_FORMAT_LOST support
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x1d3
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  precise_ip                       3
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 499  cpu 0  group_fd -1  flags 0x8 = 5
sys_perf_event_open: pid 499  cpu 1  group_fd -1  flags 0x8 = 6
sys_perf_event_open: pid 499  cpu 2  group_fd -1  flags 0x8 = 7
sys_perf_event_open: pid 499  cpu 3  group_fd -1  flags 0x8 = 9
mmap size 528384B
0x5576778fc1f8: mmap mask[0]:
0x55767790c2d8: mmap mask[0]:
0x55767791c3b8: mmap mask[0]:
0x55767792c498: mmap mask[0]:
Control descriptor is not initialized
thread_data[0x5576778cbbd0]: nr_mmaps=4, maps=0x5576778cb3d0, ow_maps=(nil)
thread_data[0x5576778cbbd0]: cpu0: maps[0] -> mmap[0]
thread_data[0x5576778cbbd0]: cpu1: maps[1] -> mmap[1]
thread_data[0x5576778cbbd0]: cpu2: maps[2] -> mmap[2]
thread_data[0x5576778cbbd0]: cpu3: maps[3] -> mmap[3]
thread_data[0x5576778cbbd0]: pollfd[0] <- event_fd=5
thread_data[0x5576778cbbd0]: pollfd[1] <- event_fd=6
thread_data[0x5576778cbbd0]: pollfd[2] <- event_fd=7
thread_data[0x5576778cbbd0]: pollfd[3] <- event_fd=9
thread_data[0x5576778cbbd0]: pollfd[4] <- non_perf_event fd=4
------------------------------------------------------------
perf_event_attr:
  type                             1 (PERF_TYPE_SOFTWARE)
  size                             136
  config                           0x9 (PERF_COUNT_SW_DUMMY)
  watermark                        1
  sample_id_all                    1
  bpf_event                        1
  { wakeup_events, wakeup_watermark } 1
------------------------------------------------------------
sys_perf_event_open: pid -1  cpu 0  group_fd -1  flags 0x8 = 10
sys_perf_event_open: pid -1  cpu 1  group_fd -1  flags 0x8 = 11
sys_perf_event_open: pid -1  cpu 2  group_fd -1  flags 0x8 = 12
sys_perf_event_open: pid -1  cpu 3  group_fd -1  flags 0x8 = 13
mmap size 528384B
0x55767799a148: mmap mask[0]:
0x5576779aa228: mmap mask[0]:
0x5576779ba308: mmap mask[0]:
0x5576779ca3e8: mmap mask[0]:
Synthesizing id index
2024-01-17T01:46:01.614080Z  INFO hotset: gups args Args { threads: 4, updates: 200000000, len: 8581545984, granularity: 8, hot_len: 1065353216, weight: 9 }
2024-01-17T01:46:16.795516Z  INFO gups: mmap init took: Ok(15.181408057s)
2024-01-17T01:46:16.795547Z  INFO gups: global memory region [0x7f0c440a0000, 0x7f0e438a0000) len 0x1ff800000
2024-01-17T01:46:16.795784Z  INFO gups: thread 0 memory region [0x7f0c440a0000, 0x7f0cc3ea0000) len 0x7fe00000
2024-01-17T01:46:16.795871Z  INFO gups: thread 1 memory region [0x7f0cc3ea0000, 0x7f0d43ca0000) len 0x7fe00000
2024-01-17T01:46:16.795949Z  INFO gups: thread 2 memory region [0x7f0d43ca0000, 0x7f0dc3aa0000) len 0x7fe00000
2024-01-17T01:46:16.796178Z  INFO gups: thread 3 memory region [0x7f0dc3aa0000, 0x7f0e438a0000) len 0x7fe00000
2024-01-17T01:46:17.796504Z  INFO gups: last second gups: 0.0300251041365446
2024-01-17T01:46:18.796631Z  INFO gups: last second gups: 0.031074211371608852
2024-01-17T01:46:19.796768Z  INFO gups: last second gups: 0.030905834109902823
2024-01-17T01:46:20.796887Z  INFO gups: last second gups: 0.030866260645121626
2024-01-17T01:46:21.797026Z  INFO gups: last second gups: 0.030445796014040585
2024-01-17T01:46:22.797157Z  INFO gups: last second gups: 0.03056599594622903
2024-01-17T01:46:23.797286Z  INFO gups: last second gups: 0.030756035424009703
2024-01-17T01:46:24.797414Z  INFO gups: last second gups: 0.03068602434937132
2024-01-17T01:46:25.797547Z  INFO gups: last second gups: 0.0303259916014081
2024-01-17T01:46:26.797680Z  INFO gups: last second gups: 0.030425952100906543
2024-01-17T01:46:27.797782Z  INFO gups: last second gups: 0.03098692876154273
2024-01-17T01:46:28.797916Z  INFO gups: last second gups: 0.030735776258890728
2024-01-17T01:46:29.798048Z  INFO gups: last second gups: 0.030515959107210982
2024-01-17T01:46:30.798186Z  INFO gups: last second gups: 0.0304858523388176
2024-01-17T01:46:31.798342Z  INFO gups: last second gups: 0.0249361348492261
2024-01-17T01:46:32.798596Z  INFO gups: last second gups: 0.0186152479553477
2024-01-17T01:46:33.798858Z  INFO gups: last second gups: 0.018505165784511116
2024-01-17T01:46:34.799130Z  INFO gups: last second gups: 0.01874489269155769
2024-01-17T01:46:35.799392Z  INFO gups: last second gups: 0.018755102030084434
2024-01-17T01:46:36.799527Z  INFO gups: last second gups: 0.01283825277798818
2024-01-17T01:46:37.799669Z  INFO gups: last second gups: 0.008348813099334546
2024-01-17T01:46:38.799796Z  INFO gups: last second gups: 0.008258937380339833
2024-01-17T01:46:39.799927Z  INFO gups: last second gups: 0.008228958575918465
2024-01-17T01:46:40.800060Z  INFO gups: last second gups: 0.008198884886064416
2024-01-17T01:46:41.800207Z  INFO gups: last second gups: 0.008238794557251945
2024-01-17T01:46:42.800354Z  INFO gups: last second gups: 0.008258796338245274
2024-01-17T01:46:43.800505Z  INFO gups: last second gups: 0.008148756516053164
2024-01-17T01:46:44.800649Z  INFO gups: last second gups: 0.008208829371678623
2024-01-17T01:46:45.800799Z  INFO gups: last second gups: 0.008248766834104602
2024-01-17T01:46:46.800944Z  INFO gups: last second gups: 0.008148812856496627
2024-01-17T01:46:47.801097Z  INFO gups: last second gups: 0.008278737575307142
2024-01-17T01:46:48.801242Z  INFO gups: last second gups: 0.00820880617690008
2024-01-17T01:46:49.801391Z  INFO gups: last second gups: 0.008098793150239925
2024-01-17T01:46:50.801537Z  INFO gups: last second gups: 0.00821879081040202
2024-01-17T01:46:51.801716Z  INFO gups: last second gups: 0.008228527217056055
2024-01-17T01:46:52.801860Z  INFO gups: last second gups: 0.008138816925294105
2024-01-17T01:46:53.802004Z  INFO gups: last second gups: 0.008078854531531392
2024-01-17T01:46:54.802156Z  INFO gups: last second gups: 0.00836872152714974
2024-01-17T01:46:55.802301Z  INFO gups: last second gups: 0.008248814505125766
2024-01-17T01:46:56.802433Z  INFO gups: last second gups: 0.008308890397540751
2024-01-17T01:46:57.802566Z  INFO gups: last second gups: 0.008288911309521873
2024-01-17T01:46:58.802697Z  INFO gups: last second gups: 0.008258911954421301
2024-01-17T01:46:59.802840Z  INFO gups: last second gups: 0.008198837314681568
2024-01-17T01:47:00.802981Z  INFO gups: last second gups: 0.00810885137309415
2024-01-17T01:47:01.803127Z  INFO gups: last second gups: 0.008118823046698212
2024-01-17T01:47:02.803273Z  INFO gups: last second gups: 0.008108801259692178
2024-01-17T01:47:03.803425Z  INFO gups: last second gups: 0.008028784080794884
2024-01-17T01:47:04.803580Z  INFO gups: last second gups: 0.00813873734814907
2024-01-17T01:47:05.803745Z  INFO gups: last second gups: 0.008098730135312242
2024-01-17T01:47:06.803901Z  INFO gups: last second gups: 0.008138670955033043
2024-01-17T01:47:07.804056Z  INFO gups: last second gups: 0.00822871892012364
2024-01-17T01:47:08.362124Z  INFO gups: iteration took 51.5655419s
2024-01-17T01:47:08.362163Z  INFO gups: iteration gups 0.015514236261715695
2024-01-17T01:47:08.362225Z  INFO hotset: warm up took: 51.566440136s
2024-01-17T01:47:08.362247Z  INFO gups: thread 0 memory region [0x7f0c440a0000, 0x7f0cc3ea0000) len 0x7fe00000
2024-01-17T01:47:08.362392Z  INFO gups: thread 1 memory region [0x7f0cc3ea0000, 0x7f0d43ca0000) len 0x7fe00000
2024-01-17T01:47:08.362454Z  INFO gups: thread 2 memory region [0x7f0d43ca0000, 0x7f0dc3aa0000) len 0x7fe00000
2024-01-17T01:47:08.362507Z  INFO gups: thread 3 memory region [0x7f0dc3aa0000, 0x7f0e438a0000) len 0x7fe00000
2024-01-17T01:47:08.804182Z  INFO gups: last second gups: 0.018227698698356238
2024-01-17T01:47:09.804359Z  INFO gups: last second gups: 0.03152438872185629
2024-01-17T01:47:10.804534Z  INFO gups: last second gups: 0.03157447099438417
2024-01-17T01:47:11.804715Z  INFO gups: last second gups: 0.031714282232056384
2024-01-17T01:47:12.804908Z  INFO gups: last second gups: 0.030714089902678657
2024-01-17T01:47:13.805101Z  INFO gups: last second gups: 0.03017416413561119
2024-01-17T01:47:14.805292Z  INFO gups: last second gups: 0.030484187884737897
2024-01-17T01:47:15.805475Z  INFO gups: last second gups: 0.030734406798978688
2024-01-17T01:47:16.805606Z  INFO gups: last second gups: 0.03066596638277781
2024-01-17T01:47:17.805781Z  INFO gups: last second gups: 0.03070458908378871
2024-01-17T01:47:18.805966Z  INFO gups: last second gups: 0.03110427569351847
2024-01-17T01:47:19.806149Z  INFO gups: last second gups: 0.030764346066951123
2024-01-17T01:47:20.806330Z  INFO gups: last second gups: 0.030744425051916487
2024-01-17T01:47:21.806515Z  INFO gups: last second gups: 0.031214245622605226
2024-01-17T01:47:22.806698Z  INFO gups: last second gups: 0.030974347801013272
2024-01-17T01:47:23.806804Z  INFO gups: last second gups: 0.01830805828395452
2024-01-17T01:47:24.807039Z  INFO gups: last second gups: 0.01816563815779374
2024-01-17T01:47:25.807266Z  INFO gups: last second gups: 0.017746025422686083
2024-01-17T01:47:26.807497Z  INFO gups: last second gups: 0.0177458441717368
2024-01-17T01:47:27.807715Z  INFO gups: last second gups: 0.017716165477711675
2024-01-17T01:47:28.807920Z  INFO gups: last second gups: 0.00756847001108185
2024-01-17T01:47:29.808099Z  INFO gups: last second gups: 0.007468675221326592
2024-01-17T01:47:30.808275Z  INFO gups: last second gups: 0.007338703507944769
2024-01-17T01:47:31.808440Z  INFO gups: last second gups: 0.007348775209031011
2024-01-17T01:47:32.808610Z  INFO gups: last second gups: 0.00797865297199739
2024-01-17T01:47:33.808788Z  INFO gups: last second gups: 0.008078554124618439
2024-01-17T01:47:34.808965Z  INFO gups: last second gups: 0.008218558431976797
2024-01-17T01:47:35.809134Z  INFO gups: last second gups: 0.00812861374621172
2024-01-17T01:47:36.809308Z  INFO gups: last second gups: 0.008208583576069614
2024-01-17T01:47:37.809475Z  INFO gups: last second gups: 0.00826861042694331
2024-01-17T01:47:38.809650Z  INFO gups: last second gups: 0.008288558271597121
2024-01-17T01:47:39.809815Z  INFO gups: last second gups: 0.008128649587444038
2024-01-17T01:47:40.809988Z  INFO gups: last second gups: 0.008148591426337452
2024-01-17T01:47:41.810148Z  INFO gups: last second gups: 0.008288672585662751
2024-01-17T01:47:42.810324Z  INFO gups: last second gups: 0.008418522212610798
2024-01-17T01:47:43.810495Z  INFO gups: last second gups: 0.008018629479905183
2024-01-17T01:47:44.810658Z  INFO gups: last second gups: 0.00818865421924503
2024-01-17T01:47:45.810827Z  INFO gups: last second gups: 0.00827861465662336
2024-01-17T01:47:46.811006Z  INFO gups: last second gups: 0.008208525469735769
2024-01-17T01:47:47.811185Z  INFO gups: last second gups: 0.008268530343148279
2024-01-17T01:47:48.811364Z  INFO gups: last second gups: 0.008248523415326376
2024-01-17T01:47:49.811541Z  INFO gups: last second gups: 0.008448500441862572
2024-01-17T01:47:50.811721Z  INFO gups: last second gups: 0.008238515518365777
2024-01-17T01:47:51.811895Z  INFO gups: last second gups: 0.008288553357339776
2024-01-17T01:47:52.812065Z  INFO gups: last second gups: 0.00814860785109168
2024-01-17T01:47:53.812233Z  INFO gups: last second gups: 0.007998656649611668
2024-01-17T01:47:54.812443Z  INFO gups: last second gups: 0.008368249512830408
2024-01-17T01:47:55.812610Z  INFO gups: last second gups: 0.008248622991375066
2024-01-17T01:47:56.812788Z  INFO gups: last second gups: 0.008268533335825425
2024-01-17T01:47:57.812952Z  INFO gups: last second gups: 0.008218651007061003
2024-01-17T01:47:58.813126Z  INFO gups: last second gups: 0.00823857030322529
2024-01-17T01:47:59.813289Z  INFO gups: last second gups: 0.008288647698839286
2024-01-17T01:48:00.284432Z  INFO gups: iteration took 51.921786785s
2024-01-17T01:48:00.284465Z  INFO gups: iteration gups 0.015407790246369504
2024-01-17T01:48:00.284565Z  INFO hotset: timed iteration took: 51.922317826s
2024-01-17T01:48:01.136344Z  INFO gups: last second gups: 0.00265294567597227
2024-01-17T01:48:01.136485Z  INFO gups: overall gups: 0.01546083003910712
2024-01-17T01:48:01.136503Z  INFO gups: overall elapsed: 103.487328685s
[ perf record: Woken up 1 times to write data ]
failed to write feature HYBRID_TOPOLOGY
[ perf record: Captured and wrote 0.012 MB perf.data, compressed (original 0.053 MB, ratio is 5.042) ]
clear@clr ~ $ sudo perf --no-pager script -F addr,phys_addr > perf.data.csv
clear@clr ~ $ head perf.data.csv
    7f0cd0c2f088        731ad088
    7f0cce27c418        703fa418
    7f0cce04cf48        701caf48
    7f0cc8222ee8        6a3a0ee8
    7f0cc7e83208        6a001208
    7f0d0788b030        a9e48030
    7f0ccf2d2960        71450960
    7f0cd10f4378        73672378
    7f0cc97351b0        6b8b31b0
    7f0cca564e50        6c6e2e50
```
</details>

<details><summary>Perf samples of local PMEM access</summary>

```bash
sudo perf record --count 1007 --phys-data --data --weight -z -vv -e MEM_LOAD_RETIRED.LOCAL_PMM:Pu -- /data/gups-hotset 4 200000000 8581545984 8 1065353216 9
Using CPUID GenuineIntel-6-6A-5
Attempting to add event pmu 'cpu' with 'MEM_LOAD_RETIRED.LOCAL_PMM,' that may result in non-fatal errors
After aliases, add event pmu 'cpu' with 'event,period,umask,' that may result in non-fatal errors
MEM_LOAD_RETIRED.LOCAL_PMM -> cpu/event=0xd1,period=0x186a3,umask=0x80/
DEBUGINFOD_URLS=
Compression enabled, disabling build id collection at the end of the session.
nr_cblocks: 0
affinity: SYS
mmap flush: 1
comp level: 1
maps__set_modules_path_dir: cannot open /lib/modules/5.15.19-htmm+ dir
Problems setting modules path maps, continuing anyway...
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x80d1
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  precise_ip                       3
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 572  cpu 0  group_fd -1  flags 0x8
sys_perf_event_open failed, error -22
decreasing precise_ip by one (2)
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x80d1
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  precise_ip                       2
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 572  cpu 0  group_fd -1  flags 0x8
sys_perf_event_open failed, error -22
decreasing precise_ip by one (1)
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x80d1
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  precise_ip                       1
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 572  cpu 0  group_fd -1  flags 0x8
sys_perf_event_open failed, error -22
decreasing precise_ip by one (0)
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x80d1
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 572  cpu 0  group_fd -1  flags 0x8
sys_perf_event_open failed, error -22
switching off PERF_FORMAT_LOST support
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x80d1
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  precise_ip                       3
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 572  cpu 0  group_fd -1  flags 0x8 = 5
sys_perf_event_open: pid 572  cpu 1  group_fd -1  flags 0x8 = 6
sys_perf_event_open: pid 572  cpu 2  group_fd -1  flags 0x8 = 7
sys_perf_event_open: pid 572  cpu 3  group_fd -1  flags 0x8 = 9
mmap size 528384B
0x55b1066ff1f8: mmap mask[0]:
0x55b10670f2d8: mmap mask[0]:
0x55b10671f3b8: mmap mask[0]:
0x55b10672f498: mmap mask[0]:
Control descriptor is not initialized
thread_data[0x55b1066cebd0]: nr_mmaps=4, maps=0x55b1066ce3d0, ow_maps=(nil)
thread_data[0x55b1066cebd0]: cpu0: maps[0] -> mmap[0]
thread_data[0x55b1066cebd0]: cpu1: maps[1] -> mmap[1]
thread_data[0x55b1066cebd0]: cpu2: maps[2] -> mmap[2]
thread_data[0x55b1066cebd0]: cpu3: maps[3] -> mmap[3]
thread_data[0x55b1066cebd0]: pollfd[0] <- event_fd=5
thread_data[0x55b1066cebd0]: pollfd[1] <- event_fd=6
thread_data[0x55b1066cebd0]: pollfd[2] <- event_fd=7
thread_data[0x55b1066cebd0]: pollfd[3] <- event_fd=9
thread_data[0x55b1066cebd0]: pollfd[4] <- non_perf_event fd=4
------------------------------------------------------------
perf_event_attr:
  type                             1 (PERF_TYPE_SOFTWARE)
  size                             136
  config                           0x9 (PERF_COUNT_SW_DUMMY)
  watermark                        1
  sample_id_all                    1
  bpf_event                        1
  { wakeup_events, wakeup_watermark } 1
------------------------------------------------------------
sys_perf_event_open: pid -1  cpu 0  group_fd -1  flags 0x8 = 10
sys_perf_event_open: pid -1  cpu 1  group_fd -1  flags 0x8 = 11
sys_perf_event_open: pid -1  cpu 2  group_fd -1  flags 0x8 = 12
sys_perf_event_open: pid -1  cpu 3  group_fd -1  flags 0x8 = 13
mmap size 528384B
0x55b10679d148: mmap mask[0]:
0x55b1067ad228: mmap mask[0]:
0x55b1067bd308: mmap mask[0]:
0x55b1067cd3e8: mmap mask[0]:
Synthesizing id index
2024-01-17T01:52:19.168558Z  INFO hotset: gups args Args { threads: 4, updates: 200000000, len: 8581545984, granularity: 8, hot_len: 1065353216, weight: 9 }
2024-01-17T01:52:26.043894Z  INFO gups: mmap init took: Ok(6.875309617s)
2024-01-17T01:52:26.043930Z  INFO gups: global memory region [0x7fadcd5df000, 0x7fafccddf000) len 0x1ff800000
2024-01-17T01:52:26.044124Z  INFO gups: thread 0 memory region [0x7fadcd5df000, 0x7fae4d3df000) len 0x7fe00000
2024-01-17T01:52:26.044209Z  INFO gups: thread 1 memory region [0x7fae4d3df000, 0x7faecd1df000) len 0x7fe00000
2024-01-17T01:52:26.044314Z  INFO gups: thread 2 memory region [0x7faecd1df000, 0x7faf4cfdf000) len 0x7fe00000
2024-01-17T01:52:26.044693Z  INFO gups: thread 3 memory region [0x7faf4cfdf000, 0x7fafccddf000) len 0x7fe00000
2024-01-17T01:52:27.044581Z  INFO gups: last second gups: 0.03155683544898434
2024-01-17T01:52:28.044722Z  INFO gups: last second gups: 0.03185477049789167
2024-01-17T01:52:29.044865Z  INFO gups: last second gups: 0.0319554987484463
2024-01-17T01:52:30.044993Z  INFO gups: last second gups: 0.03214580500459271
2024-01-17T01:52:31.045126Z  INFO gups: last second gups: 0.03178590104913021
2024-01-17T01:52:32.045255Z  INFO gups: last second gups: 0.031965762043199834
2024-01-17T01:52:33.045389Z  INFO gups: last second gups: 0.03177575272578766
2024-01-17T01:52:34.045520Z  INFO gups: last second gups: 0.03156584180009383
2024-01-17T01:52:35.045647Z  INFO gups: last second gups: 0.03174598606926739
2024-01-17T01:52:36.045774Z  INFO gups: last second gups: 0.0315559557256023
2024-01-17T01:52:37.045905Z  INFO gups: last second gups: 0.03189589477507118
2024-01-17T01:52:38.046031Z  INFO gups: last second gups: 0.03188591034502506
2024-01-17T01:52:39.046162Z  INFO gups: last second gups: 0.031745882781224456
2024-01-17T01:52:40.046298Z  INFO gups: last second gups: 0.032055678317560586
2024-01-17T01:52:41.046452Z  INFO gups: last second gups: 0.025845950792427203
2024-01-17T01:52:42.046786Z  INFO gups: last second gups: 0.019213554563345517
2024-01-17T01:52:43.047061Z  INFO gups: last second gups: 0.018954830638588244
2024-01-17T01:52:44.047401Z  INFO gups: last second gups: 0.018593671476569908
2024-01-17T01:52:45.047901Z  INFO gups: last second gups: 0.013143456622124872
2024-01-17T01:52:46.048059Z  INFO gups: last second gups: 0.00813870799638299
2024-01-17T01:52:47.048253Z  INFO gups: last second gups: 0.008168417883645371
2024-01-17T01:52:48.048432Z  INFO gups: last second gups: 0.008098549736008924
2024-01-17T01:52:49.048604Z  INFO gups: last second gups: 0.008228587661670925
2024-01-17T01:52:50.048815Z  INFO gups: last second gups: 0.008148270879881393
2024-01-17T01:52:51.048997Z  INFO gups: last second gups: 0.008168520411392323
2024-01-17T01:52:52.049165Z  INFO gups: last second gups: 0.008368584320064912
2024-01-17T01:52:53.049357Z  INFO gups: last second gups: 0.008118454790144428
2024-01-17T01:52:54.049520Z  INFO gups: last second gups: 0.007808726576296117
2024-01-17T01:52:55.049698Z  INFO gups: last second gups: 0.00806855777757149
2024-01-17T01:52:56.049893Z  INFO gups: last second gups: 0.008228399872447204
2024-01-17T01:52:57.050071Z  INFO gups: last second gups: 0.008168541049557302
2024-01-17T01:52:58.050267Z  INFO gups: last second gups: 0.008318379088969102
2024-01-17T01:52:59.050476Z  INFO gups: last second gups: 0.008478220158763851
2024-01-17T01:53:00.050651Z  INFO gups: last second gups: 0.008148555749978874
2024-01-17T01:53:01.050841Z  INFO gups: last second gups: 0.008058488775482956
2024-01-17T01:53:02.051080Z  INFO gups: last second gups: 0.008318017941094906
2024-01-17T01:53:03.051245Z  INFO gups: last second gups: 0.008298623872397735
2024-01-17T01:53:04.051431Z  INFO gups: last second gups: 0.007988510318572813
2024-01-17T01:53:05.051621Z  INFO gups: last second gups: 0.008258426472453645
2024-01-17T01:53:06.051800Z  INFO gups: last second gups: 0.00817853813537953
2024-01-17T01:53:07.051984Z  INFO gups: last second gups: 0.008208491722479961
2024-01-17T01:53:08.052171Z  INFO gups: last second gups: 0.00826846333916227
2024-01-17T01:53:09.052354Z  INFO gups: last second gups: 0.008248473306572633
2024-01-17T01:53:10.052538Z  INFO gups: last second gups: 0.008088515830141812
2024-01-17T01:53:11.052722Z  INFO gups: last second gups: 0.008208484286959444
2024-01-17T01:53:12.052894Z  INFO gups: last second gups: 0.008108630833357895
2024-01-17T01:53:13.053177Z  INFO gups: last second gups: 0.007457890654954497
2024-01-17T01:53:14.053408Z  INFO gups: last second gups: 0.0074182697108447545
2024-01-17T01:53:15.053604Z  INFO gups: last second gups: 0.007588520481338794
2024-01-17T01:53:16.053805Z  INFO gups: last second gups: 0.007718445551376626
2024-01-17T01:53:17.053995Z  INFO gups: last second gups: 0.007498574108640372
2024-01-17T01:53:17.066978Z  INFO gups: iteration took 51.022027945s
2024-01-17T01:53:17.067002Z  INFO gups: iteration gups 0.01567950221152269
2024-01-17T01:53:17.067116Z  INFO hotset: warm up took: 51.022992398s
2024-01-17T01:53:17.067134Z  INFO gups: thread 0 memory region [0x7fadcd5df000, 0x7fae4d3df000) len 0x7fe00000
2024-01-17T01:53:17.067198Z  INFO gups: thread 1 memory region [0x7fae4d3df000, 0x7faecd1df000) len 0x7fe00000
2024-01-17T01:53:17.067231Z  INFO gups: thread 2 memory region [0x7faecd1df000, 0x7faf4cfdf000) len 0x7fe00000
2024-01-17T01:53:17.067323Z  INFO gups: thread 3 memory region [0x7faf4cfdf000, 0x7fafccddf000) len 0x7fe00000
2024-01-17T01:53:18.054130Z  INFO gups: last second gups: 0.029995927872815697
2024-01-17T01:53:19.054320Z  INFO gups: last second gups: 0.030714097119102873
2024-01-17T01:53:20.054503Z  INFO gups: last second gups: 0.030394416728013563
2024-01-17T01:53:21.054688Z  INFO gups: last second gups: 0.030644374887901202
2024-01-17T01:53:22.054785Z  INFO gups: last second gups: 0.030756966040598858
2024-01-17T01:53:23.054977Z  INFO gups: last second gups: 0.03070420378322662
2024-01-17T01:53:24.055162Z  INFO gups: last second gups: 0.030394304502462197
2024-01-17T01:53:25.055342Z  INFO gups: last second gups: 0.030064577011727777
2024-01-17T01:53:26.055528Z  INFO gups: last second gups: 0.030024450971044885
2024-01-17T01:53:27.055715Z  INFO gups: last second gups: 0.030794305332292122
2024-01-17T01:53:28.055909Z  INFO gups: last second gups: 0.030683983208416643
2024-01-17T01:53:29.056094Z  INFO gups: last second gups: 0.030964272631384192
2024-01-17T01:53:30.056286Z  INFO gups: last second gups: 0.03007421191717442
2024-01-17T01:53:31.056472Z  INFO gups: last second gups: 0.030504336686868062
2024-01-17T01:53:32.056664Z  INFO gups: last second gups: 0.03004422384774415
2024-01-17T01:53:33.056887Z  INFO gups: last second gups: 0.02022554487877846
2024-01-17T01:53:34.057260Z  INFO gups: last second gups: 0.018663022903539602
2024-01-17T01:53:35.057614Z  INFO gups: last second gups: 0.019083234211059577
2024-01-17T01:53:36.057972Z  INFO gups: last second gups: 0.018823267519552815
2024-01-17T01:53:37.058115Z  INFO gups: last second gups: 0.008258822283683524
2024-01-17T01:53:38.058265Z  INFO gups: last second gups: 0.008208772427336126
2024-01-17T01:53:39.058414Z  INFO gups: last second gups: 0.008188773166193014
2024-01-17T01:53:40.058569Z  INFO gups: last second gups: 0.008348696701655297
2024-01-17T01:53:41.058716Z  INFO gups: last second gups: 0.008208807695308679
2024-01-17T01:53:42.058880Z  INFO gups: last second gups: 0.008258635855046735
2024-01-17T01:53:43.059028Z  INFO gups: last second gups: 0.008258783960132143
2024-01-17T01:53:44.059198Z  INFO gups: last second gups: 0.00792865897833504
2024-01-17T01:53:45.059346Z  INFO gups: last second gups: 0.008228794744891306
2024-01-17T01:53:46.059492Z  INFO gups: last second gups: 0.008198796318314916
2024-01-17T01:53:47.059645Z  INFO gups: last second gups: 0.008108752208992575
2024-01-17T01:53:48.059795Z  INFO gups: last second gups: 0.008248768648562449
2024-01-17T01:53:49.059944Z  INFO gups: last second gups: 0.008268765142613602
2024-01-17T01:53:50.060088Z  INFO gups: last second gups: 0.008288800154733602
2024-01-17T01:53:51.060231Z  INFO gups: last second gups: 0.008238827977287262
2024-01-17T01:53:52.060372Z  INFO gups: last second gups: 0.00821884005866484
2024-01-17T01:53:53.060515Z  INFO gups: last second gups: 0.00806885946671438
2024-01-17T01:53:54.060664Z  INFO gups: last second gups: 0.007988795944652814
2024-01-17T01:53:55.060808Z  INFO gups: last second gups: 0.008208815878309553
2024-01-17T01:53:56.060943Z  INFO gups: last second gups: 0.00829888771667711
2024-01-17T01:53:57.061077Z  INFO gups: last second gups: 0.008258882730085392
2024-01-17T01:53:58.061214Z  INFO gups: last second gups: 0.008138879292599167
2024-01-17T01:53:59.061362Z  INFO gups: last second gups: 0.0083387980456497
2024-01-17T01:54:00.061514Z  INFO gups: last second gups: 0.008288720992329836
2024-01-17T01:54:01.061664Z  INFO gups: last second gups: 0.00834876256311042
2024-01-17T01:54:02.061817Z  INFO gups: last second gups: 0.008368707787830481
2024-01-17T01:54:03.061961Z  INFO gups: last second gups: 0.008038853120941795
2024-01-17T01:54:04.062116Z  INFO gups: last second gups: 0.008038744492807625
2024-01-17T01:54:05.062272Z  INFO gups: last second gups: 0.00822871543169139
2024-01-17T01:54:06.062422Z  INFO gups: last second gups: 0.008228755664025998
2024-01-17T01:54:07.062572Z  INFO gups: last second gups: 0.008118791785762818
2024-01-17T01:54:08.062716Z  INFO gups: last second gups: 0.008068819475229038
2024-01-17T01:54:08.732013Z  INFO gups: iteration took 51.664581289s
2024-01-17T01:54:08.732047Z  INFO gups: iteration gups 0.015484495955265382
2024-01-17T01:54:08.732211Z  INFO hotset: timed iteration took: 51.66507645s
2024-01-17T01:54:09.432366Z  INFO gups: last second gups: 0.0028182391599236175
2024-01-17T01:54:09.432485Z  INFO gups: overall gups: 0.015581388965273505
2024-01-17T01:54:09.432499Z  INFO gups: overall elapsed: 102.686609234s
[ perf record: Woken up 1 times to write data ]
failed to write feature HYBRID_TOPOLOGY
[ perf record: Captured and wrote 0.028 MB perf.data, compressed (original 0.133 MB, ratio is 5.003) ]
clear@clr ~ $ sudo perf --no-pager script -F addr,phys_addr > perf.data.csv
clear@clr ~ $ head perf.data.csv
    7faedce53ac0       4b5b63ac0
    7faecf6c6580       4c2c13580
    7faedaac6508       4b7fd6508
    7faed13538f8       4c10a08f8
    7faed255f898       4c02ac898
    7faed831c640       4ba069640
    7faedbce5c48       4b69f5c48
    7faee25ac9a8       4b02bc9a8
    7faed2ef1de0       4bf43ede0
    7faecd5fdb70       4c534ab70
```
</details>

### Our kernel is working
To make sure that we can indeed collect DRAM and PMEM access samples via PEBS,
we run perf with
`MEM_LOAD_L3_MISS_RETIRED.LOCAL_DRAM` and `MEM_LOAD_RETIRED.LOCAL_PMM`
in a VM with our kernel to profile `gups` workload.

<details><summary>System physical memory range</summary>

```dmesg
[    0.109283] ACPI: SRAT: Node 1 PXM 1 [mem 0x00000000-0xbfffffff]
[    0.109289] ACPI: SRAT: Node 1 PXM 1 [mem 0x100000000-0x2bfffffff]
[    0.109290] ACPI: SRAT: Node 2 PXM 2 [mem 0x2c0000000-0x53fffffff]
[    0.109308] NUMA: Initialized distance table, cnt=3
[    0.109314] NUMA: Node 1 [mem 0x00000000-0xbfffffff] + [mem 0x100000000-0x2bfffffff] -> [mem 0x00000000-0x2bfffffff]
[    0.109325] NODE_DATA(1) allocated [mem 0x2bffdd000-0x2bfffffff]
[    0.109497] NODE_DATA(2) allocated [mem 0x53b7dc000-0x53b7fefff]
[    0.109736] Zone ranges:
[    0.109741]   DMA      [mem 0x0000000000001000-0x0000000000ffffff]
[    0.109746]   Normal   [mem 0x0000000001000000-0x000000053fffffff]
[    0.109748]   Device   empty
[    0.109749] Movable zone start for each node
[    0.109752] Early memory node ranges
[    0.109754]   node   1: [mem 0x0000000000001000-0x000000000009ffff]
[    0.109758]   node   1: [mem 0x0000000000100000-0x00000000bfffffff]
[    0.109759]   node   1: [mem 0x0000000100000000-0x00000002bfffffff]
[    0.109760]   node   2: [mem 0x00000002c0000000-0x000000053fffffff]
[    0.109761] Initializing node 0 as memoryless
[    0.109920] Initmem setup node 0 as memoryless
[    0.109927] Initmem setup node 1 [mem 0x0000000000001000-0x00000002bfffffff]
[    0.109932] Initmem setup node 2 [mem 0x00000002c0000000-0x000000053fffffff]
```
</details>

<details><summary>Perf samples of local DRAM access</summary>

```bash
clear@clr ~ $ sudo perf record --count 1007 --phys-data --data --weight -z -vv -e MEM_LOAD_L3_MISS_RETIRED.LOCAL_DRAM:Pu -- /data/gups-hotset 4 200000000 8581545984 8 1065353216 9
Using CPUID GenuineIntel-6-6A-5
Attempting to add event pmu 'cpu' with 'MEM_LOAD_L3_MISS_RETIRED.LOCAL_DRAM,' that may result in non-fatal errors
After aliases, add event pmu 'cpu' with 'event,period,umask,' that may result in non-fatal errors
MEM_LOAD_L3_MISS_RETIRED.LOCAL_DRAM -> cpu/event=0xd3,period=0x186a7,umask=0x1/
DEBUGINFOD_URLS=
Compression enabled, disabling build id collection at the end of the session.
nr_cblocks: 0
affinity: SYS
mmap flush: 1
comp level: 1
maps__set_modules_path_dir: cannot open /lib/modules/6.6.0+ dir
Problems setting modules path maps, continuing anyway...
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x1d3
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  precise_ip                       3
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 781  cpu 0  group_fd -1  flags 0x8 = 5
sys_perf_event_open: pid 781  cpu 1  group_fd -1  flags 0x8 = 6
sys_perf_event_open: pid 781  cpu 2  group_fd -1  flags 0x8 = 7
sys_perf_event_open: pid 781  cpu 3  group_fd -1  flags 0x8 = 9
mmap size 528384B
0x564c3c50c1c8: mmap mask[0]:
0x564c3c51c2a8: mmap mask[0]:
0x564c3c52c388: mmap mask[0]:
0x564c3c53c468: mmap mask[0]:
Control descriptor is not initialized
thread_data[0x564c3c4dbba0]: nr_mmaps=4, maps=0x564c3c4db3a0, ow_maps=(nil)
thread_data[0x564c3c4dbba0]: cpu0: maps[0] -> mmap[0]
thread_data[0x564c3c4dbba0]: cpu1: maps[1] -> mmap[1]
thread_data[0x564c3c4dbba0]: cpu2: maps[2] -> mmap[2]
thread_data[0x564c3c4dbba0]: cpu3: maps[3] -> mmap[3]
thread_data[0x564c3c4dbba0]: pollfd[0] <- event_fd=5
thread_data[0x564c3c4dbba0]: pollfd[1] <- event_fd=6
thread_data[0x564c3c4dbba0]: pollfd[2] <- event_fd=7
thread_data[0x564c3c4dbba0]: pollfd[3] <- event_fd=9
thread_data[0x564c3c4dbba0]: pollfd[4] <- non_perf_event fd=4
------------------------------------------------------------
perf_event_attr:
  type                             1 (PERF_TYPE_SOFTWARE)
  size                             136
  config                           0x9 (PERF_COUNT_SW_DUMMY)
  watermark                        1
  sample_id_all                    1
  bpf_event                        1
  { wakeup_events, wakeup_watermark } 1
------------------------------------------------------------
sys_perf_event_open: pid -1  cpu 0  group_fd -1  flags 0x8 = 10
sys_perf_event_open: pid -1  cpu 1  group_fd -1  flags 0x8 = 11
sys_perf_event_open: pid -1  cpu 2  group_fd -1  flags 0x8 = 12
sys_perf_event_open: pid -1  cpu 3  group_fd -1  flags 0x8 = 13
mmap size 528384B
0x564c3c5aa118: mmap mask[0]:
0x564c3c5ba1f8: mmap mask[0]:
0x564c3c5ca2d8: mmap mask[0]:
0x564c3c5da3b8: mmap mask[0]:
Synthesizing id index
2024-01-17T01:15:04.733785Z  INFO hotset: gups args Args { threads: 4, updates: 200000000, len: 8581545984, granularity: 8, hot_len: 1065353216, weight: 9 }
2024-01-17T01:15:20.216851Z  INFO gups: mmap init took: Ok(15.483043838s)
2024-01-17T01:15:20.216895Z  INFO gups: global memory region [0x7f787ea00000, 0x7f7a7e200000) len 0x1ff800000
2024-01-17T01:15:20.217108Z  INFO gups: thread 0 memory region [0x7f787ea00000, 0x7f78fe800000) len 0x7fe00000
2024-01-17T01:15:20.217177Z  INFO gups: thread 1 memory region [0x7f78fe800000, 0x7f797e600000) len 0x7fe00000
2024-01-17T01:15:20.217354Z  INFO gups: thread 2 memory region [0x7f797e600000, 0x7f79fe400000) len 0x7fe00000
2024-01-17T01:15:20.217768Z  INFO gups: thread 3 memory region [0x7f79fe400000, 0x7f7a7e200000) len 0x7fe00000
2024-01-17T01:15:21.220595Z  INFO gups: last second gups: 0.021117997095930412
2024-01-17T01:15:22.221379Z  INFO gups: last second gups: 0.021342752622806972
2024-01-17T01:15:23.222302Z  INFO gups: last second gups: 0.02024134719373403
2024-01-17T01:15:24.223105Z  INFO gups: last second gups: 0.021023101168772314
2024-01-17T01:15:25.224383Z  INFO gups: last second gups: 0.021322747013232942
2024-01-17T01:15:26.225420Z  INFO gups: last second gups: 0.020748518685131948
2024-01-17T01:15:27.228953Z  INFO gups: last second gups: 0.020397915805335523
2024-01-17T01:15:28.229824Z  INFO gups: last second gups: 0.02088182181030494
2024-01-17T01:15:29.230584Z  INFO gups: last second gups: 0.02059429339144773
2024-01-17T01:15:30.231235Z  INFO gups: last second gups: 0.020276822518858265
2024-01-17T01:15:31.232526Z  INFO gups: last second gups: 0.020893039830333335
2024-01-17T01:15:32.235879Z  INFO gups: last second gups: 0.020401632843040102
2024-01-17T01:15:33.237254Z  INFO gups: last second gups: 0.020351980797125194
2024-01-17T01:15:34.238646Z  INFO gups: last second gups: 0.02050144161483918
2024-01-17T01:15:35.241480Z  INFO gups: last second gups: 0.020282532203425973
2024-01-17T01:15:36.242721Z  INFO gups: last second gups: 0.020754265955471172
2024-01-17T01:15:37.244965Z  INFO gups: last second gups: 0.0204740293956356
2024-01-17T01:15:38.254652Z  INFO gups: last second gups: 0.020798568690834608
2024-01-17T01:15:39.258407Z  INFO gups: last second gups: 0.020582665963552436
2024-01-17T01:15:40.259015Z  INFO gups: last second gups: 0.020507535519910997
2024-01-17T01:15:41.259740Z  INFO gups: last second gups: 0.02037524561226905
2024-01-17T01:15:42.263058Z  INFO gups: last second gups: 0.020790999247517324
2024-01-17T01:15:43.267901Z  INFO gups: last second gups: 0.020978388594500737
2024-01-17T01:15:44.272219Z  INFO gups: last second gups: 0.020939669490526967
2024-01-17T01:15:45.272517Z  INFO gups: last second gups: 0.020943705411226067
2024-01-17T01:15:46.273356Z  INFO gups: last second gups: 0.020972413247312853
2024-01-17T01:15:47.277163Z  INFO gups: last second gups: 0.021019961807964692
2024-01-17T01:15:48.277998Z  INFO gups: last second gups: 0.01597665235372104
2024-01-17T01:15:49.281985Z  INFO gups: last second gups: 0.014960343774118076
2024-01-17T01:15:50.282830Z  INFO gups: last second gups: 0.01544696646757332
2024-01-17T01:15:51.286209Z  INFO gups: last second gups: 0.015049144486234246
2024-01-17T01:15:52.286704Z  INFO gups: last second gups: 0.012124000377793048
2024-01-17T01:15:53.286855Z  INFO gups: last second gups: 0.008508702065569513
2024-01-17T01:15:54.287010Z  INFO gups: last second gups: 0.009008617321379045
2024-01-17T01:15:55.287207Z  INFO gups: last second gups: 0.008998222868979822
2024-01-17T01:15:56.287380Z  INFO gups: last second gups: 0.008648491054523262
2024-01-17T01:15:57.287664Z  INFO gups: last second gups: 0.008767517477426266
2024-01-17T01:15:58.288009Z  INFO gups: last second gups: 0.008896923897459377
2024-01-17T01:15:59.288195Z  INFO gups: last second gups: 0.008698394398171619
2024-01-17T01:16:00.288350Z  INFO gups: last second gups: 0.008738643823649077
2024-01-17T01:16:01.288467Z  INFO gups: last second gups: 0.009198930697897815
2024-01-17T01:16:02.288709Z  INFO gups: last second gups: 0.009087776729949678
2024-01-17T01:16:03.288859Z  INFO gups: last second gups: 0.008638713971928428
2024-01-17T01:16:04.289037Z  INFO gups: last second gups: 0.00920836924384876
2024-01-17T01:16:05.289211Z  INFO gups: last second gups: 0.009288379001385058
2024-01-17T01:16:06.289391Z  INFO gups: last second gups: 0.009338325656886371
2024-01-17T01:16:07.289550Z  INFO gups: last second gups: 0.009078544237274465
2024-01-17T01:16:08.289702Z  INFO gups: last second gups: 0.009098616819172532
2024-01-17T01:16:09.289850Z  INFO gups: last second gups: 0.009208633908368328
2024-01-17T01:16:10.290002Z  INFO gups: last second gups: 0.009148614286839815
2024-01-17T01:16:10.775436Z  INFO gups: iteration took 50.557384785s
2024-01-17T01:16:10.775466Z  INFO gups: iteration gups 0.015823603285693568
2024-01-17T01:16:10.775679Z  INFO hotset: warm up took: 50.5585723s
2024-01-17T01:16:10.775704Z  INFO gups: thread 0 memory region [0x7f787ea00000, 0x7f78fe800000) len 0x7fe00000
2024-01-17T01:16:10.775774Z  INFO gups: thread 1 memory region [0x7f78fe800000, 0x7f797e600000) len 0x7fe00000
2024-01-17T01:16:10.775807Z  INFO gups: thread 2 memory region [0x7f797e600000, 0x7f79fe400000) len 0x7fe00000
2024-01-17T01:16:10.775889Z  INFO gups: thread 3 memory region [0x7f79fe400000, 0x7f7a7e200000) len 0x7fe00000
2024-01-17T01:16:11.292507Z  INFO gups: last second gups: 0.014314142000727528
2024-01-17T01:16:12.292678Z  INFO gups: last second gups: 0.019656672184369204
2024-01-17T01:16:13.292927Z  INFO gups: last second gups: 0.018715295779398778
2024-01-17T01:16:14.293195Z  INFO gups: last second gups: 0.019474802447873897
2024-01-17T01:16:15.293328Z  INFO gups: last second gups: 0.019667399576428005
2024-01-17T01:16:16.294234Z  INFO gups: last second gups: 0.019901979672897255
2024-01-17T01:16:17.294690Z  INFO gups: last second gups: 0.01971101445810507
2024-01-17T01:16:18.294873Z  INFO gups: last second gups: 0.019966327673216377
2024-01-17T01:16:19.295963Z  INFO gups: last second gups: 0.020118045157201902
2024-01-17T01:16:20.299137Z  INFO gups: last second gups: 0.019667613131422727
2024-01-17T01:16:21.300167Z  INFO gups: last second gups: 0.020578788907224813
2024-01-17T01:16:22.301636Z  INFO gups: last second gups: 0.02001060068561051
2024-01-17T01:16:23.301969Z  INFO gups: last second gups: 0.01977341606542611
2024-01-17T01:16:24.302253Z  INFO gups: last second gups: 0.020454198473415994
2024-01-17T01:16:25.305168Z  INFO gups: last second gups: 0.019882039520513088
2024-01-17T01:16:26.308747Z  INFO gups: last second gups: 0.019819037895178238
2024-01-17T01:16:27.310071Z  INFO gups: last second gups: 0.019973577034834926
2024-01-17T01:16:28.311692Z  INFO gups: last second gups: 0.020446851890864162
2024-01-17T01:16:29.314478Z  INFO gups: last second gups: 0.02026355455166529
2024-01-17T01:16:30.315333Z  INFO gups: last second gups: 0.019713187706430508
2024-01-17T01:16:31.318800Z  INFO gups: last second gups: 0.020080336553456295
2024-01-17T01:16:32.318933Z  INFO gups: last second gups: 0.020057321384843698
2024-01-17T01:16:33.319619Z  INFO gups: last second gups: 0.020485942116315
2024-01-17T01:16:34.322624Z  INFO gups: last second gups: 0.02024915731651276
2024-01-17T01:16:35.323456Z  INFO gups: last second gups: 0.021432227710950793
2024-01-17T01:16:36.324278Z  INFO gups: last second gups: 0.021442316435866075
2024-01-17T01:16:37.325126Z  INFO gups: last second gups: 0.02169160337287107
2024-01-17T01:16:38.325583Z  INFO gups: last second gups: 0.021140354163763576
2024-01-17T01:16:39.327696Z  INFO gups: last second gups: 0.021185228578517703
2024-01-17T01:16:40.327944Z  INFO gups: last second gups: 0.01678583832067683
2024-01-17T01:16:41.328579Z  INFO gups: last second gups: 0.015280308693415831
2024-01-17T01:16:42.328727Z  INFO gups: last second gups: 0.015647666385273633
2024-01-17T01:16:43.329030Z  INFO gups: last second gups: 0.014815507626961846
2024-01-17T01:16:44.329173Z  INFO gups: last second gups: 0.010298536135178137
2024-01-17T01:16:45.329320Z  INFO gups: last second gups: 0.009288630177841784
2024-01-17T01:16:46.329463Z  INFO gups: last second gups: 0.009368654211559817
2024-01-17T01:16:47.329603Z  INFO gups: last second gups: 0.009228706126172404
2024-01-17T01:16:48.329749Z  INFO gups: last second gups: 0.009188705265484567
2024-01-17T01:16:49.329914Z  INFO gups: last second gups: 0.009568371759746015
2024-01-17T01:16:50.330063Z  INFO gups: last second gups: 0.009738553464484045
2024-01-17T01:16:51.330212Z  INFO gups: last second gups: 0.009638563073378457
2024-01-17T01:16:52.330366Z  INFO gups: last second gups: 0.00964852467303782
2024-01-17T01:16:53.330631Z  INFO gups: last second gups: 0.008857645071050831
2024-01-17T01:16:54.330781Z  INFO gups: last second gups: 0.009418582851768375
2024-01-17T01:16:55.330937Z  INFO gups: last second gups: 0.009238566839603305
2024-01-17T01:16:56.331082Z  INFO gups: last second gups: 0.009518623711716146
2024-01-17T01:16:57.331230Z  INFO gups: last second gups: 0.009598575888091211
2024-01-17T01:16:58.331378Z  INFO gups: last second gups: 0.00947860138497264
2024-01-17T01:16:59.332464Z  INFO gups: last second gups: 0.009449725115437757
2024-01-17T01:17:00.332616Z  INFO gups: last second gups: 0.008618691673985197
2024-01-17T01:17:00.442235Z  INFO gups: iteration took 49.666056109s
2024-01-17T01:17:00.442253Z  INFO gups: iteration gups 0.016107580562553098
2024-01-17T01:17:00.445842Z  INFO hotset: timed iteration took: 49.670138923s
2024-01-17T01:17:01.332786Z  INFO gups: last second gups: 0.000609896308479114
2024-01-17T01:17:01.333049Z  INFO gups: overall gups: 0.01596432916020334
2024-01-17T01:17:01.333063Z  INFO gups: overall elapsed: 100.223440894s
[ perf record: Woken up 36 times to write data ]
failed to write feature HYBRID_TOPOLOGY
[ perf record: Captured and wrote 2.072 MB perf.data, compressed (original 8.842 MB, ratio is 4.271) ]
clear@clr ~ $  sudo perf --no-pager script -F addr,phys_addr > perf.data.csv
clear@clr ~ $  head perf.data.csv
    7f790bf91ec8       289fe3ec8
    7f79096dff28       287731f28
    7f7902961be8       2805b3be8
    7f790e2c3b90       28c315b90
    7f79102bb260       28e30d260
    7f790110f158       27ed61158
    7f78fe833710       27c485710
    7f7903e87cc0       281ed9cc0
    7f79078c3478       285915478
    7f7902eb14d8       280f034d8
```
</details>

<details><summary>Perf samples of local PMEM access</summary>

```bash
clear@clr ~ $ sudo perf record --count 1007 --phys-data --data --weight -z -vv -e MEM_LOAD_RETIRED.LOCAL_PMM:Pu -- /data/gups-hotset 4 200000000 8581545984 8 1065353216 9
Using CPUID GenuineIntel-6-6A-5
Attempting to add event pmu 'cpu' with 'MEM_LOAD_RETIRED.LOCAL_PMM,' that may result in non-fatal errors
After aliases, add event pmu 'cpu' with 'event,period,umask,' that may result in non-fatal errors
MEM_LOAD_RETIRED.LOCAL_PMM -> cpu/event=0xd1,period=0x186a3,umask=0x80/
DEBUGINFOD_URLS=
Compression enabled, disabling build id collection at the end of the session.
nr_cblocks: 0
affinity: SYS
mmap flush: 1
comp level: 1
maps__set_modules_path_dir: cannot open /lib/modules/6.6.0+ dir
Problems setting modules path maps, continuing anyway...
------------------------------------------------------------
perf_event_attr:
  type                             4 (PERF_TYPE_RAW)
  size                             136
  config                           0x80d1
  { sample_period, sample_freq }   1007
  sample_type                      IP|TID|TIME|ADDR|DATA_SRC|PHYS_ADDR|WEIGHT_STRUCT
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  exclude_kernel                   1
  exclude_hv                       1
  mmap                             1
  comm                             1
  enable_on_exec                   1
  task                             1
  precise_ip                       3
  mmap_data                        1
  sample_id_all                    1
  exclude_guest                    1
  mmap2                            1
  comm_exec                        1
  ksymbol                          1
  bpf_event                        1
------------------------------------------------------------
sys_perf_event_open: pid 20205  cpu 0  group_fd -1  flags 0x8 = 5
sys_perf_event_open: pid 20205  cpu 1  group_fd -1  flags 0x8 = 6
sys_perf_event_open: pid 20205  cpu 2  group_fd -1  flags 0x8 = 7
sys_perf_event_open: pid 20205  cpu 3  group_fd -1  flags 0x8 = 9
mmap size 528384B
0x55574635e1c8: mmap mask[0]:
0x55574636e2a8: mmap mask[0]:
0x55574637e388: mmap mask[0]:
0x55574638e468: mmap mask[0]:
Control descriptor is not initialized
thread_data[0x55574632dba0]: nr_mmaps=4, maps=0x55574632d3a0, ow_maps=(nil)
thread_data[0x55574632dba0]: cpu0: maps[0] -> mmap[0]
thread_data[0x55574632dba0]: cpu1: maps[1] -> mmap[1]
thread_data[0x55574632dba0]: cpu2: maps[2] -> mmap[2]
thread_data[0x55574632dba0]: cpu3: maps[3] -> mmap[3]
thread_data[0x55574632dba0]: pollfd[0] <- event_fd=5
thread_data[0x55574632dba0]: pollfd[1] <- event_fd=6
thread_data[0x55574632dba0]: pollfd[2] <- event_fd=7
thread_data[0x55574632dba0]: pollfd[3] <- event_fd=9
thread_data[0x55574632dba0]: pollfd[4] <- non_perf_event fd=4
------------------------------------------------------------
perf_event_attr:
  type                             1 (PERF_TYPE_SOFTWARE)
  size                             136
  config                           0x9 (PERF_COUNT_SW_DUMMY)
  watermark                        1
  sample_id_all                    1
  bpf_event                        1
  { wakeup_events, wakeup_watermark } 1
------------------------------------------------------------
sys_perf_event_open: pid -1  cpu 0  group_fd -1  flags 0x8 = 10
sys_perf_event_open: pid -1  cpu 1  group_fd -1  flags 0x8 = 11
sys_perf_event_open: pid -1  cpu 2  group_fd -1  flags 0x8 = 12
sys_perf_event_open: pid -1  cpu 3  group_fd -1  flags 0x8 = 13
mmap size 528384B
0x5557463fc118: mmap mask[0]:
0x55574640c1f8: mmap mask[0]:
0x55574641c2d8: mmap mask[0]:
0x55574642c3b8: mmap mask[0]:
Synthesizing id index
2024-01-17T01:28:56.350970Z  INFO hotset: gups args Args { threads: 4, updates: 200000000, len: 8581545984, granularity: 8, hot_len: 1065353216, weight: 9 }
2024-01-17T01:29:03.855732Z  INFO gups: mmap init took: Ok(7.504305217s)
2024-01-17T01:29:03.855778Z  INFO gups: global memory region [0x7f2fddc00000, 0x7f31dd400000) len 0x1ff800000
2024-01-17T01:29:03.859539Z  INFO gups: thread 0 memory region [0x7f2fddc00000, 0x7f305da00000) len 0x7fe00000
2024-01-17T01:29:03.859630Z  INFO gups: thread 1 memory region [0x7f305da00000, 0x7f30dd800000) len 0x7fe00000
2024-01-17T01:29:03.859782Z  INFO gups: thread 2 memory region [0x7f30dd800000, 0x7f315d600000) len 0x7fe00000
2024-01-17T01:29:03.860022Z  INFO gups: thread 3 memory region [0x7f315d600000, 0x7f31dd400000) len 0x7fe00000
2024-01-17T01:29:04.875573Z  INFO gups: last second gups: 0.01751173517218398
2024-01-17T01:29:05.875838Z  INFO gups: last second gups: 0.01710488644288324
2024-01-17T01:29:06.876008Z  INFO gups: last second gups: 0.017416996682760012
2024-01-17T01:29:07.876245Z  INFO gups: last second gups: 0.01746585519536944
2024-01-17T01:29:08.876413Z  INFO gups: last second gups: 0.017536996017804125
2024-01-17T01:29:09.876624Z  INFO gups: last second gups: 0.01748636638552419
2024-01-17T01:29:10.876791Z  INFO gups: last second gups: 0.017587048471177372
2024-01-17T01:29:11.876978Z  INFO gups: last second gups: 0.01761676165163967
2024-01-17T01:29:12.877753Z  INFO gups: last second gups: 0.01661708470325609
2024-01-17T01:29:13.877909Z  INFO gups: last second gups: 0.015857549342609495
2024-01-17T01:29:14.878109Z  INFO gups: last second gups: 0.016696626763798284
2024-01-17T01:29:15.878710Z  INFO gups: last second gups: 0.016690035314555655
2024-01-17T01:29:16.879007Z  INFO gups: last second gups: 0.016485034608665662
2024-01-17T01:29:17.881488Z  INFO gups: last second gups: 0.01645924997494179
2024-01-17T01:29:18.881772Z  INFO gups: last second gups: 0.016705126513211543
2024-01-17T01:29:19.882028Z  INFO gups: last second gups: 0.016215841317767328
2024-01-17T01:29:20.882318Z  INFO gups: last second gups: 0.016505319091505648
2024-01-17T01:29:21.882477Z  INFO gups: last second gups: 0.016637348821827894
2024-01-17T01:29:22.882669Z  INFO gups: last second gups: 0.016506776375134924
2024-01-17T01:29:23.882944Z  INFO gups: last second gups: 0.016045603167774362
2024-01-17T01:29:24.883141Z  INFO gups: last second gups: 0.016526770189828573
2024-01-17T01:29:25.883357Z  INFO gups: last second gups: 0.015976538363432796
2024-01-17T01:29:26.883636Z  INFO gups: last second gups: 0.015675596160716992
2024-01-17T01:29:27.883817Z  INFO gups: last second gups: 0.016497090094775273
2024-01-17T01:29:28.884068Z  INFO gups: last second gups: 0.01624591330801554
2024-01-17T01:29:29.884333Z  INFO gups: last second gups: 0.01660560398185228
2024-01-17T01:29:30.886496Z  INFO gups: last second gups: 0.015676074544893295
2024-01-17T01:29:31.886699Z  INFO gups: last second gups: 0.015166967061596693
2024-01-17T01:29:32.886949Z  INFO gups: last second gups: 0.016465844416049653
2024-01-17T01:29:33.887147Z  INFO gups: last second gups: 0.016866706522550873
2024-01-17T01:29:34.887358Z  INFO gups: last second gups: 0.017676166781824026
2024-01-17T01:29:35.887555Z  INFO gups: last second gups: 0.017696577481915
2024-01-17T01:29:36.893503Z  INFO gups: last second gups: 0.01775439638866591
2024-01-17T01:29:37.893851Z  INFO gups: last second gups: 0.0175938110250957
2024-01-17T01:29:38.896324Z  INFO gups: last second gups: 0.017446880558107055
2024-01-17T01:29:39.903459Z  INFO gups: last second gups: 0.01765401422828752
2024-01-17T01:29:40.904389Z  INFO gups: last second gups: 0.017723533419570545
2024-01-17T01:29:41.909225Z  INFO gups: last second gups: 0.017913426238160446
2024-01-17T01:29:42.910293Z  INFO gups: last second gups: 0.01801065874980308
2024-01-17T01:29:43.914457Z  INFO gups: last second gups: 0.017198455491459767
2024-01-17T01:29:44.915422Z  INFO gups: last second gups: 0.018002524283584526
2024-01-17T01:29:45.916006Z  INFO gups: last second gups: 0.017389855332264382
2024-01-17T01:29:46.916190Z  INFO gups: last second gups: 0.01412751446161568
2024-01-17T01:29:47.916627Z  INFO gups: last second gups: 0.010515381360591132
2024-01-17T01:29:48.916821Z  INFO gups: last second gups: 0.009468138639688546
2024-01-17T01:29:49.917021Z  INFO gups: last second gups: 0.006888618156974947
2024-01-17T01:29:50.917225Z  INFO gups: last second gups: 0.0047990258313494166
2024-01-17T01:29:51.917437Z  INFO gups: last second gups: 0.004719002520842157
2024-01-17T01:29:52.917617Z  INFO gups: last second gups: 0.004829138404442433
2024-01-17T01:29:53.917820Z  INFO gups: last second gups: 0.004739024533407117
2024-01-17T01:29:54.917996Z  INFO gups: last second gups: 0.004959140496646542
2024-01-17T01:29:55.918179Z  INFO gups: last second gups: 0.004889099833605936
2024-01-17T01:29:56.918379Z  INFO gups: last second gups: 0.004719052343503639
2024-01-17T01:29:57.919827Z  INFO gups: last second gups: 0.0048629713238489605
2024-01-17T01:29:58.920003Z  INFO gups: last second gups: 0.0049391152414694495
2024-01-17T01:29:59.667238Z  INFO gups: iteration took 55.806239209s
2024-01-17T01:29:59.667303Z  INFO gups: iteration gups 0.014335314676982966
2024-01-17T01:29:59.669264Z  INFO hotset: warm up took: 55.809712045s
2024-01-17T01:29:59.669365Z  INFO gups: thread 0 memory region [0x7f2fddc00000, 0x7f305da00000) len 0x7fe00000
2024-01-17T01:29:59.669482Z  INFO gups: thread 1 memory region [0x7f305da00000, 0x7f30dd800000) len 0x7fe00000
2024-01-17T01:29:59.669593Z  INFO gups: thread 2 memory region [0x7f30dd800000, 0x7f315d600000) len 0x7fe00000
2024-01-17T01:29:59.669641Z  INFO gups: thread 3 memory region [0x7f315d600000, 0x7f31dd400000) len 0x7fe00000
2024-01-17T01:29:59.922520Z  INFO gups: last second gups: 0.007780432779531072
2024-01-17T01:30:00.923250Z  INFO gups: last second gups: 0.01735735508003742
2024-01-17T01:30:01.924221Z  INFO gups: last second gups: 0.017423118757658822
2024-01-17T01:30:02.925824Z  INFO gups: last second gups: 0.01730219739314631
2024-01-17T01:30:03.928848Z  INFO gups: last second gups: 0.017158145236842153
2024-01-17T01:30:04.929865Z  INFO gups: last second gups: 0.017801908401143443
2024-01-17T01:30:05.940540Z  INFO gups: last second gups: 0.017770231361654495
2024-01-17T01:30:06.941120Z  INFO gups: last second gups: 0.017479937569058746
2024-01-17T01:30:07.955198Z  INFO gups: last second gups: 0.017868401295301317
2024-01-17T01:30:08.955786Z  INFO gups: last second gups: 0.017619766668575856
2024-01-17T01:30:09.957262Z  INFO gups: last second gups: 0.01773373167941136
2024-01-17T01:30:10.960808Z  INFO gups: last second gups: 0.01775701917355399
2024-01-17T01:30:11.961831Z  INFO gups: last second gups: 0.017552056339731274
2024-01-17T01:30:12.962091Z  INFO gups: last second gups: 0.016495770401990077
2024-01-17T01:30:13.963149Z  INFO gups: last second gups: 0.015903144209742662
2024-01-17T01:30:14.974553Z  INFO gups: last second gups: 0.016610549835578692
2024-01-17T01:30:15.986532Z  INFO gups: last second gups: 0.01651213260514835
2024-01-17T01:30:16.989454Z  INFO gups: last second gups: 0.0164742211058651
2024-01-17T01:30:17.990107Z  INFO gups: last second gups: 0.016576723057298626
2024-01-17T01:30:18.994638Z  INFO gups: last second gups: 0.016604732656412737
2024-01-17T01:30:20.002516Z  INFO gups: last second gups: 0.0163313989866103
2024-01-17T01:30:21.007489Z  INFO gups: last second gups: 0.01684615091183588
2024-01-17T01:30:22.020539Z  INFO gups: last second gups: 0.016751427540959832
2024-01-17T01:30:23.032051Z  INFO gups: last second gups: 0.016717525693494942
2024-01-17T01:30:24.032470Z  INFO gups: last second gups: 0.01632320562888903
2024-01-17T01:30:25.048487Z  INFO gups: last second gups: 0.016426860066956907
2024-01-17T01:30:26.057534Z  INFO gups: last second gups: 0.01600528306347516
2024-01-17T01:30:27.068535Z  INFO gups: last second gups: 0.01609293045181451
2024-01-17T01:30:28.077545Z  INFO gups: last second gups: 0.01657067173459651
2024-01-17T01:30:29.077817Z  INFO gups: last second gups: 0.016195573555400425
2024-01-17T01:30:30.078697Z  INFO gups: last second gups: 0.01639563100015651
2024-01-17T01:30:31.080165Z  INFO gups: last second gups: 0.016256110004481305
2024-01-17T01:30:32.083968Z  INFO gups: last second gups: 0.016009124216546138
2024-01-17T01:30:33.086171Z  INFO gups: last second gups: 0.01646397381239128
2024-01-17T01:30:34.090960Z  INFO gups: last second gups: 0.01754570495911646
2024-01-17T01:30:35.091925Z  INFO gups: last second gups: 0.017603031751176803
2024-01-17T01:30:36.092201Z  INFO gups: last second gups: 0.01772509629893361
2024-01-17T01:30:37.093796Z  INFO gups: last second gups: 0.017671809240698676
2024-01-17T01:30:38.097013Z  INFO gups: last second gups: 0.01789243170931795
2024-01-17T01:30:39.098100Z  INFO gups: last second gups: 0.01758089132921428
2024-01-17T01:30:40.114497Z  INFO gups: last second gups: 0.01763092851482576
2024-01-17T01:30:41.125631Z  INFO gups: last second gups: 0.014765547547709134
2024-01-17T01:30:42.126370Z  INFO gups: last second gups: 0.011681381861947856
2024-01-17T01:30:43.126576Z  INFO gups: last second gups: 0.009278118007987024
2024-01-17T01:30:44.126771Z  INFO gups: last second gups: 0.009398141526911194
2024-01-17T01:30:45.126963Z  INFO gups: last second gups: 0.009548163468045903
2024-01-17T01:30:46.127355Z  INFO gups: last second gups: 0.009676210225179416
2024-01-17T01:30:47.127547Z  INFO gups: last second gups: 0.00944817847622339
2024-01-17T01:30:48.127721Z  INFO gups: last second gups: 0.006348892861016218
2024-01-17T01:30:49.127892Z  INFO gups: last second gups: 0.00476917583872331
2024-01-17T01:30:50.128187Z  INFO gups: last second gups: 0.004408697437116112
2024-01-17T01:30:51.128480Z  INFO gups: last second gups: 0.004788601584679226
2024-01-17T01:30:52.128773Z  INFO gups: last second gups: 0.004718612100467054
2024-01-17T01:30:53.129080Z  INFO gups: last second gups: 0.004638574366638434
2024-01-17T01:30:54.129266Z  INFO gups: last second gups: 0.004809114319843057
2024-01-17T01:30:55.129504Z  INFO gups: last second gups: 0.004868839341733521
2024-01-17T01:30:55.196282Z  INFO gups: iteration took 55.526414456s
2024-01-17T01:30:55.196312Z  INFO gups: iteration gups 0.014407557337128845
2024-01-17T01:30:55.196417Z  INFO hotset: timed iteration took: 55.527058603s
2024-01-17T01:30:56.129764Z  INFO gups: last second gups: 0.00021994181175439864
2024-01-17T01:30:56.130045Z  INFO gups: overall gups: 0.014371345219295685
2024-01-17T01:30:56.130059Z  INFO gups: overall elapsed: 111.332653665s
[ perf record: Woken up 45 times to write data ]
failed to write feature HYBRID_TOPOLOGY
[ perf record: Captured and wrote 2.715 MB perf.data, compressed (original 11.252 MB, ratio is 4.148) ]
clear@clr ~ $ sudo perf --no-pager script -F addr,phys_addr > perf.data.csv
clear@clr ~ $ head perf.data.csv
    7f30679f7860       49ca46860
    7f3068264aa8       49cb9baa8
    7f306b20db90       418669b90
    7f30632a3ce0       45f3d1ce0
    7f305f57d050       41225f050
    7f306304ce90       45f267e90
    7f30cc6468f8       4e46728f8
    7f30653cf9f8       48ccfa9f8
    7f3068781550       4134c5550
    7f30dc4d70c8       4a03960c8
```
</details>


### Debugging
We know that perf ring buffer are filled by the
`perf_event_output{,_forward,_backword}()`-series of functions.
We can insert these code to inspect every enqueued sample.
```c
	pr_info_ratelimited("%s: event=0x%px data=0x%px regs=0x%px id=0x%lx type=0x%lx pid=0x%x addr=0x%px phys_addr=0x%px\n",
			    __func__, event, data, regs, data->id, data->type, data->tid_entry.pid, data->addr, data->phys_addr);
```
However, in Memtis' kernel,
these code output nothing for kernel events created by memtis' sampler nor for userspace perf invocations.

Perf ring buffer are filled by draining PEBS buffers during context swithc.
For icelake-x, that would be `intel_pmu_drain_pebs_icl()`.
The call chain for entering this function is:
```c
=> __schedule()
=> context_switch()
=> prepare_task_switch()
=> perf_event_task_sched_out()
    // guarded by static_branch_unlikely(&perf_sched_events)
    // enabled by perf_event_alloc()=>account_event(event)
=> __perf_event_task_sched_out()
    // guarded by __this_cpu_read(perf_sched_cb_usages)
    // enabled by perf_sched_cb_inc()
=> perf_pmu_sched_task(sched_in=false)
=> __perf_pmu_sched_task()
    // set in init_hw_perf_events() via perf_pmu_register(&pmu, "cpu", PERF_TYPE_RAW)
=> pmu->sched_task()
=> x86_pmu_sched_task()
    // set in x86_pmu_static_call_update() via changing __SCK__x86_pmu_sched_task()
=> static_call_cond(x86_pmu_sched_task)()
    // set in intel_pmu_init() via x86_pmu = intel_pmu;
=> x86_pmu.sched_task()
=> intel_pmu_sched_task()
=> intel_pmu_pebs_sched_task()
=> intel_pmu_drain_pebs_buffer()
    // set in intel_ds_init() via x86_pmu.drain_pebs = intel_pmu_drain_pebs_icl;
=> x86_pmu.drain_pebs()
=> intel_pmu_drain_pebs_icl()

// `PERF_TYPE_*` are mapped to `struct pmu` using a radix tree, specifically `pmu_idr`
// For example, PERF_TYPE_RAW are mapped to cpu pmu
// via `perf_pmu_register(&pmu, "cpu", PERF_TYPE_RAW)`
// during architecture perf initialization `init_hw_perf_events()`
```

