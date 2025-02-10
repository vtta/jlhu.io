+++
+++
## Memtis

### Migration workflow

Memtis每个node会启动一个`kmigraterd`线程. DRAM node是demotion线程. PMEM node是promotion线程.

demotion的目标是为promotion让出可用空间, 并且保持上层可用空间不至于太低. 前者通过计算promotion所需页数, 后者通过计算上层可用内存低于max水位的页数加上一个额外headroom得到. demotion的页数量为这前后二者总和. demotion会首先尝试demote文件页及inactive anon页. 只有在急需为promotion或其他内存分配请求让出空间时才会demote active anon页.

promotion的目标是尽量将下层所有的active anon页移到上层, 并保持下层可用空间不至于太多. 前者会计算所有的active anon页, 后者会计算下层可用内存超过max水位的页数得到. promotion的页数量为这二之中较小者.

`kmigraterd`线程会轮训的检测下层node中的active anon页以及各层的水位情况, 并据此做迁移.

下面为当前memtis设计执行GUPS过程中的统计信息. **memtis设计在此场景下无效对根本原因是热度的区分没有做好.** 可以看到promotion能够找到候选页, 但是由于上层内存不足无法完成promotion. 上层内存不足又是由demotion无法完成引起. **demotion虽然能找到很多备选页, 但是备选页都被视为warm的页**, 从而没有被demote掉.

```
# hotset (old) reverse=true working_set=14G
# initial 
dram portion per gb: [0.998046875, 1.0, 1.0, 1.0, 1.0, 1.0, 0.5167274475097656, 0.001827239990234375, 0.001331329345703125, 0.00156402587890625, 0.001407623291015625, 0.0013427734375, 0.0013427734375, 0.009319305419921875]
# final
dram portion per gb: [0.9980583190917969, 1.0, 1.0, 1.0, 1.0, 1.0, 0.5169754028320313, 0.0041046142578125, 0.029205322265625, 0.00428009033203125, 0.01242828369140625, 0.0601959228515625, 0.046566009521484375, 0.01285552978515625]
gups: overall gups: 0.011044578247848475
gups: overall elapsed: 144.867460223s

htmm_nr_promotion_scanned 67753654
htmm_nr_promotion_scanned_anon_inactive 0
htmm_nr_promotion_scanned_anon_active 67753654
htmm_nr_promotion_scanned_file_inactive 0
htmm_nr_promotion_scanned_file_active 0
htmm_nr_promotion_tried 67753654
htmm_nr_promotion_tried_anon_inactive 0
htmm_nr_promotion_tried_anon_active 67753654
htmm_nr_promotion_tried_file_inactive 0
htmm_nr_promotion_tried_file_active 0
htmm_nr_promotion_tried_failure 67351569
htmm_nr_promotion_tried_failure_nomem 67351569
htmm_nr_promoted 402083 # 1570M
htmm_nr_promoted_anon_inactive 0
htmm_nr_promoted_anon_active 402083
htmm_nr_promoted_file_inactive 0
htmm_nr_promoted_file_active 0
htmm_nr_demotion_scanned 590966624
htmm_nr_demotion_scanned_anon_inactive 590965028
htmm_nr_demotion_scanned_anon_active 0
htmm_nr_demotion_scanned_file_inactive 1215
htmm_nr_demotion_scanned_file_active 381
htmm_nr_demotion_scanned_filtered 590201710
htmm_nr_demotion_scanned_filtered_warm 590201710
htmm_nr_demotion_tried 764914
htmm_nr_demotion_tried_anon_inactive 763318
htmm_nr_demotion_tried_anon_active 0
htmm_nr_demotion_tried_file_inactive 1215
htmm_nr_demotion_tried_file_active 381
htmm_nr_demoted 3740
htmm_nr_demoted_anon_inactive 2165
htmm_nr_demoted_anon_active 0
htmm_nr_demoted_file_inactive 1215
htmm_nr_demoted_file_active 360

[   75.683454] __cooling: increasing cooling_clock=1
[   75.683666] cooling happens ts=75683666863
# no other cooling found

[   90.838759] __adjust_active_threshold: warm_threshold: 1 -> 0
[   90.839265] __adjust_active_threshold: active_threshold: 1 -> 1
[   90.839563] __adjust_active_threshold: hotness_hg: 280 0 0 0 0 0 0 0 3316 64137 8223 51 0 1 0 13
[   90.839572] __adjust_active_threshold: accumulated hotness_hg: 280 280 280 280 280 280 280 280 3596 67733 75956 76007 76007 76008 76008 76021
[   90.839824] __adjust_active_threshold: ebp_hotness_hg: 280 0 0 0 0 0 0 0 3316 64137 8223 51 0 1 0 13
[   90.840150] __adjust_active_threshold: accumulated ebp_hotness_hg: 280 280 280 280 280 280 280 280 3596 67733 75956 76007 76007 76008 76008 76021
# ... 10 other occurances
[  222.935831] __adjust_active_threshold: warm_threshold: 1 -> 0
[  222.936254] __adjust_active_threshold: active_threshold: 1 -> 1
[  222.936424] __adjust_active_threshold: hotness_hg: 0 0 0 0 0 0 0 0 3218 137552 99345 123774 9384 2 1 18
[  222.936429] __adjust_active_threshold: accumulated hotness_hg: 0 0 0 0 0 0 0 0 3218 140770 240115 363889 373273 373275 373276 373294
[  222.936692] __adjust_active_threshold: ebp_hotness_hg: 0 0 0 0 0 0 0 0 3218 137552 99345 123774 9384 2 1 18
[  222.936973] __adjust_active_threshold: accumulated ebp_hotness_hg: 0 0 0 0 0 0 0 0 3218 140770 240115 363889 373273 373275 373276 373294
```

#### Histogram/cooling

从dmesg中可以看出histogram将bucket 1就划为了warm. Cooling甚至在几分钟内只发生了1次.

### 验证实验

#### 访问模式

所有页都被划为warm的原因有可能是gups-hotset的访问模式. 虽然内存被分为hot/cold两区, 但是cold区所有页也都会被访问到. 这导致了cold区被划为warm.

实验: 更改访问模式为zipf, 检查`htmm_nr_demotion_scanned_filtered_warm`以及`warm_threshold`.

```
# hotset reverse=true working_set=14G
# initial
dram portion per gb:  [0.998046875, 1.0, 1.0, 1.0, 1.0, 1.0, 0.5129852294921875, 0.00376129150390625, 0.001922607421875, 0.0014190673828125, 0.0043487548828125, 0.0013885498046875, 0.001422882080078125, 0.09565597768321368]
# final
dram portion per gb: [0.9980583190917969, 1.0, 1.0, 1.0, 1.0, 1.0, 0.51312255859375, 0.05799102783203125, 0.002445220947265625, 0.002696990966796875, 0.00954437255859375, 0.001678466796875, 0.001922607421875, 0.0956751633660897]
GUPS: final 0.017615 elapsed 42.296961727s

htmm_nr_promotion_scanned 58733146
htmm_nr_promotion_scanned_anon_inactive 0
htmm_nr_promotion_scanned_anon_active 58733146
htmm_nr_promotion_scanned_file_inactive 0
htmm_nr_promotion_scanned_file_active 0
htmm_nr_promotion_tried 58733146
htmm_nr_promotion_tried_anon_inactive 0
htmm_nr_promotion_tried_anon_active 58733146
htmm_nr_promotion_tried_file_inactive 0
htmm_nr_promotion_tried_file_active 0
htmm_nr_promotion_tried_failure 58547086
htmm_nr_promotion_tried_failure_nomem 58547086
htmm_nr_promoted 186060 # 726M
htmm_nr_promoted_anon_inactive 0
htmm_nr_promoted_anon_active 186060
htmm_nr_promoted_file_inactive 0
htmm_nr_promoted_file_active 0
htmm_nr_demotion_scanned 500951579
htmm_nr_demotion_scanned_anon_inactive 500948999
htmm_nr_demotion_scanned_anon_active 0
htmm_nr_demotion_scanned_file_inactive 1797
htmm_nr_demotion_scanned_file_active 783
htmm_nr_demotion_scanned_filtered 500948489
htmm_nr_demotion_scanned_filtered_warm 500948489
htmm_nr_demotion_tried 3090
htmm_nr_demotion_tried_anon_inactive 510
htmm_nr_demotion_tried_anon_active 0
htmm_nr_demotion_tried_file_inactive 1797
htmm_nr_demotion_tried_file_active 783
htmm_nr_demoted 3090
htmm_nr_demoted_anon_inactive 510
htmm_nr_demoted_anon_active 0
htmm_nr_demoted_file_inactive 1797
htmm_nr_demoted_file_active 783

[  203.678604] __adjust_active_threshold: warm_threshold: 1 -> 0
[  203.679064] __adjust_active_threshold: active_threshold: 1 -> 1
[  203.679227] __adjust_active_threshold: hotness_hg: 0 2481425 0 0 0 0 0 3121 187289 47715 145798 77569 351 4 1 70
[  203.679234] __adjust_active_threshold: accumulated hotness_hg: 0 2481425 2481425 2481425 2481425 2481425 2481425 2484546 2671835 2719550 2865348 2942917 2943268 2943272 2943273 2943343
[  203.679450] __adjust_active_threshold: ebp_hotness_hg: 0 2481425 0 0 0 0 0 3121 187289 47715 145798 77569 351 4 1 70
[  203.679783] __adjust_active_threshold: accumulated ebp_hotness_hg: 0 2481425 2481425 2481425 2481425 2481425 2481425 2484546 2671835 2719550 2865348 2942917 2943268 2943272 2943273 2943343
```

```
# zipf reverse=true working_set=14G
# initial
dram portion per gb: [0.998046875, 1.0, 1.0, 1.0, 1.0, 1.0, 0.5154342651367188, 0.002285003662109375, 0.002166748046875, 0.0023040771484375, 0.003475189208984375, 0.003017425537109375, 0.003070831298828125, 0.011718615100667278]
# final
dram portion per gb: [0.998046875, 1.0, 1.0, 1.0, 1.0, 1.0, 0.5154342651367188, 0.002285003662109375, 0.057636260986328125, 0.07538223266601563, 0.003955841064453125, 0.0084991455078125, 0.003070831298828125, 0.011718615100667278]
GUPS: final 0.023816 elapsed 31.283282587s

htmm_nr_promotion_scanned 45205597
htmm_nr_promotion_scanned_anon_inactive 0
htmm_nr_promotion_scanned_anon_active 45205597
htmm_nr_promotion_scanned_file_inactive 0
htmm_nr_promotion_scanned_file_active 0
htmm_nr_promotion_tried 45205597
htmm_nr_promotion_tried_anon_inactive 0
htmm_nr_promotion_tried_anon_active 45205597
htmm_nr_promotion_tried_file_inactive 0
htmm_nr_promotion_tried_file_active 0
htmm_nr_promotion_tried_failure 44725307
htmm_nr_promotion_tried_failure_nomem 44725307
htmm_nr_promoted 480290 #1,876M
htmm_nr_promoted_anon_inactive 0
htmm_nr_promoted_anon_active 480290
htmm_nr_promoted_file_inactive 0
htmm_nr_promoted_file_active 0
htmm_nr_demotion_scanned 399802623
htmm_nr_demotion_scanned_anon_inactive 399800158
htmm_nr_demotion_scanned_anon_active 0
htmm_nr_demotion_scanned_file_inactive 1764
htmm_nr_demotion_scanned_file_active 701
htmm_nr_demotion_scanned_filtered 399799648
htmm_nr_demotion_scanned_filtered_warm 399799648
htmm_nr_demotion_tried 2975
htmm_nr_demotion_tried_anon_inactive 510
htmm_nr_demotion_tried_anon_active 0
htmm_nr_demotion_tried_file_inactive 1764
htmm_nr_demotion_tried_file_active 701
htmm_nr_demoted 2974
htmm_nr_demoted_anon_inactive 510
htmm_nr_demoted_anon_active 0
htmm_nr_demoted_file_inactive 1763
htmm_nr_demoted_file_active 701

[  166.586600] __adjust_active_threshold: warm_threshold: 1 -> 0
[  166.586836] __adjust_active_threshold: active_threshold: 1 -> 1
[  166.586967] __adjust_active_threshold: hotness_hg: 0 0 0 0 0 0 0 0 3182 234172 39497 11199 5223 1092 29 85
[  166.586973] __adjust_active_threshold: accumulated hotness_hg: 0 0 0 0 0 0 0 0 3182 237354 276851 288050 293273 294365 294394 294479
[  166.587180] __adjust_active_threshold: ebp_hotness_hg: 0 0 0 0 0 0 0 0 3182 234172 39497 11199 5223 1092 29 85
[  166.587437] __adjust_active_threshold: accumulated ebp_hotness_hg: 0 0 0 0 0 0 0 0 3182 237354 276851 288050 293273 294365 294394 294479
```

从以上zipf实验结果看: `warm_threshold=0`且`htmm_nr_demotion_scanned_filtered_warm -> htmm_nr_demotion_scanned`. 可以说明Memtis的coldness识别有问题, 所有的页都被识别为hot. 即使对于zipf这种skewness很高的, 也无法区分出基本没有被访问的页.

### 测试DRAM充足时的Promotion

```
# random working_set=7G prefer=PMEM updates=8e8
# initial
dram portion per gb: [0.1170654296875, 0.13534927368164063, 0.145233154296875, 0.14971160888671875, 0.15735626220703125, 0.16620254516601563, 0.17886812145304687]
# final
dram portion per gb: [0.5795097351074219, 0.5802383422851563, 0.583343505859375, 0.5819625854492188, 0.586517333984375, 0.5951385498046875, 0.6007919849891217]
GUPS: final 0.033527 elapsed 22.22238231s

htmm_nr_promotion_scanned 1162383
htmm_nr_promotion_scanned_anon_inactive 0
htmm_nr_promotion_scanned_anon_active 1162383
htmm_nr_promotion_scanned_file_inactive 0
htmm_nr_promotion_scanned_file_active 0
htmm_nr_promotion_tried 1162383
htmm_nr_promotion_tried_anon_inactive 0
htmm_nr_promotion_tried_anon_active 1162383
htmm_nr_promotion_tried_file_inactive 0
htmm_nr_promotion_tried_file_active 0
htmm_nr_promotion_tried_failure 0
htmm_nr_promotion_tried_failure_nomem 0
htmm_nr_promoted 1162382
htmm_nr_promoted_anon_inactive 0
htmm_nr_promoted_anon_active 1162382
htmm_nr_promoted_file_inactive 0
htmm_nr_promoted_file_active 0
htmm_nr_demotion_scanned 0
htmm_nr_demotion_scanned_anon_inactive 0
htmm_nr_demotion_scanned_anon_active 0
htmm_nr_demotion_scanned_file_inactive 0
htmm_nr_demotion_scanned_file_active 0
htmm_nr_demotion_scanned_filtered 0
htmm_nr_demotion_scanned_filtered_warm 0
htmm_nr_demotion_tried 0
htmm_nr_demotion_tried_anon_inactive 0
htmm_nr_demotion_tried_anon_active 0
htmm_nr_demotion_tried_file_inactive 0
htmm_nr_demotion_tried_file_active 0
htmm_nr_demoted 0
htmm_nr_demoted_anon_inactive 0
htmm_nr_demoted_anon_active 0
htmm_nr_demoted_file_inactive 0
htmm_nr_demoted_file_active 0

[  147.730103] __adjust_active_threshold: warm_threshold: 0 -> 0
[  147.730357] __adjust_active_threshold: active_threshold: 1 -> 1
[  147.730472] __adjust_active_threshold: hotness_hg: 0 0 0 0 0 0 0 0 0 674794 400637 25749 12 1 9 74
[  147.730478] __adjust_active_threshold: accumulated hotness_hg: 0 0 0 0 0 0 0 0 0 674794 1075431 1101180 1101192 1101193 1101202 1101276
[  147.730631] __adjust_active_threshold: ebp_hotness_hg: 0 0 0 0 0 0 0 0 0 674794 400637 25749 12 1 9 74
[  147.730842] __adjust_active_threshold: accumulated ebp_hotness_hg: 0 0 0 0 0 0 0 0 0 674794 1075431 1101180 1101192 1101193 1101202 1101276
```

上述结果中GUPS快结束时有60%的内存都被promte到了DRAM, 说明promotion本身时没有问题. 之前限制promotion发生的根本原因还是coldness识别有问题, 无法为promotion腾出足够的DRAM.

运行十倍的work在结束时有99%内存都被promote到了DRAM.

```
# random working_set=7G prefer=PMEM updates=8e9
dram portion per gb: [0.1100006103515625, 0.11920166015625, 0.12438583374023438, 0.13076400756835938, 0.14511871337890625, 0.15521621704101563, 0.1627176136080211
4]
dram portion per gb: [0.9974136352539063, 1.0, 1.0, 1.0, 0.7765998840332031, 0.7979850769042969, 0.9999961628634249]
GUPS: final 0.055139 elapsed 135.12475487s
```

## TPP

通过preferred node方式分配内存会override migration.

```
# random working_set=7G prefer=PMEM updates=8e9 scan_period_min_ms=1000 scan_period_max_ms=1000
dram portion per gb: [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
dram portion per gb: [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
GUPS: final 0.022739 elapsed 327.661303501s
```

### 修改mpol实现提高numa placement优先级

```diff
diff --git a/mm/mempolicy.c b/mm/mempolicy.c
index 8921cb5603a6..ab841b9f1560 100644
--- a/mm/mempolicy.c
+++ b/mm/mempolicy.c
@@ -2425,6 +2425,9 @@ int mpol_misplaced(struct page *page, struct vm_area_struc
t *vma, unsigned long
                break;
 
        case MPOL_PREFERRED:
+               // Allow heterogeneous management to override preferred nodes
+               if (pol->flags & MPOL_F_MORON && numa_promotion_tiered_enabled &
& !node_is_toptier(curnid))
+                       break;
                if (node_isset(curnid, pol->nodes))
                        goto out;
                polnid = first_node(pol->nodes);

```

```
# random working_set=7G prefer=PMEM updates=8e8
dram portion per gb: [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
dram portion per gb: [0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0]
GUPS: final 0.022783 elapsed 32.702536121s
```

修改后仍没有promotion. 检查vmstat后发现, 原因是hinting fault没有触发. 检查代码后发现preferred policy不会设置`MPOL_F_MOF`这个标记, 导致`task_numa_work()`直接跳过了相关VMA的扫描.

```
numa_pte_updates 0
numa_huge_pte_updates 0
numa_hint_faults 0
numa_hint_faults_local 0
numa_pages_migrated 0
```

### 为preferred policy添加NUMA scan支持

```diff
diff --git a/mm/mempolicy.c b/mm/mempolicy.c
index 8921cb5603a6..fe5bad981980 100644
--- a/mm/mempolicy.c
+++ b/mm/mempolicy.c
@@ -283,6 +283,7 @@ static struct mempolicy *mpol_new(unsigned short mode, unsigned short flags,
 
                        mode = MPOL_LOCAL;
                }
+               flags |= MPOL_F_MOF | MPOL_F_MORON;
```

修改后NUMA hinting fault能正常触发

```
# random working_set=7G prefer=PMEM updates=8e8
dram portion per gb: [0.7578010559082031, 0.7893753051757813, 0.8205680847167969, 0.8520355224609375, 0.8519401550292969, 0.7945823669433594, 0.6763835755206035]
dram portion per gb: [0.9751319885253906, 0.9772148132324219, 0.9771957397460938, 0.9767799377441406, 0.9752311706542969, 0.9534950256347656, 0.9057599257130359]
GUPS: final 0.022783 elapsed 32.702536121s
```

```
numa_pte_updates 2000501
numa_huge_pte_updates 0
numa_hint_faults 56374746
numa_hint_faults_local 0
numa_pages_migrated 1776389 # 6,939 MiB
pgpromote_candidate 2000430
pgpromote_candidate_demoted 0
pgpromote_candidate_anon 2000430
pgpromote_candidate_file 0
pgpromote_tried 1776405
pgpromote_file 0
pgpromote_anon 1776389
pgmigrate_success 1776389
pgmigrate_fail 16
pgmigrate_fail_dst_node_full 224019
pgmigrate_fail_numa_isolate 224025
pgmigrate_fail_nomem 0
pgmigrate_fail_refcount 162
```

### 对比用满内存的表现

```
# random working_set=14G prefer=PMEM updates=8e8
dram portion per gb: [0.040557861328125, 0.041576385498046875, 0.041004180908203125, 0.040767669677734375, 0.04198455810546875, 0.041233062744140625, 0.041744232177734375, 0.4420356750488281, 1.0, 0.9998817443847656, 1.0, 1.0, 0.9995193481445313, 1.0]
dram portion per gb: [0.040691375732421875, 0.0416717529296875, 0.041107177734375, 0.040771484375, 0.04198455810546875, 0.041233062744140625, 0.041744232177734375, 0.4420356750488281, 1.0, 0.9998817443847656, 1.0, 1.0, 0.9995193481445313, 1.0]
GUPS: final 0.020198 elapsed 36.888040019s
```

```
numa_pte_updates 7591805
numa_huge_pte_updates 0
numa_hint_faults 63457614
numa_hint_faults_local 0
numa_pages_migrated 82072
pgpromote_candidate 7559115
pgpromote_candidate_demoted 0
pgpromote_candidate_anon 7559115
pgpromote_candidate_file 0
pgpromote_tried 82074
pgpromote_file 0
pgpromote_anon 82072  # 320 MiB
pgmigrate_success 117331
pgmigrate_fail 176
pgmigrate_fail_dst_node_full 7477038
pgmigrate_fail_numa_isolate 7477041
pgmigrate_fail_nomem 0
pgmigrate_fail_refcount 20

pgactivate 192000791
pgdeactivate 188307718
pglazyfree 0
pgfault 67836955
pgmajfault 10003
pglazyfreed 0
pgrefill 188623861
pgreuse 16878
pgsteal_kswapd 160640
pgsteal_direct 0
pgdemote_kswapd 0
pgdemote_direct 0
pgdemote_file 0
pgdemote_anon 0
pgdemote_failed 0
pgscan_kswapd 232739803
pgscan_direct 0
pgscan_direct_throttle 0
pgscan_anon 231697046
pgscan_file 1042757
pgsteal_anon 0
pgsteal_file 160640
```

总结: promotion受限于demotion, demotion受限于coldness identification.

### 待验证BUG

~~[Memtis在分配大于cgroup DRAM限制的内存时](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/mempolicy.c#L2124)会[回退到PMEM](https://github.com/cosmoss-jigu/memtis/blob/cb79fa4d73dcf33fed1aa6a7d9e1fa47a8fd088a/linux/mm/mempolicy.c#L2129). 这与其文中说的优先分配DRAM不符. ("Memtis allocates pages on the fast tier whenever available.")~~
