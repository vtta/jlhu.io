+++
+++


### 观察: dmesg中cooling只在程序开始时发生一次

- 触发: 每收到2M个sample执行一次cooling (sec 4.2.2)
  - 观察: 收到了20,592,866个PEBS sample
  - 文中所指的sample是经过过滤后的, 即从虚拟地址能找到有htmm flag的page和其关联`pginfo`
    - 初步观察发现只有巨页会被设置htmm flag (`SetPageHtmm()`)  [link](#htmm-flag)
  - 即`pebs_sample_collected=20592866`但是`htmm_nr_sampled=0`
- Base page: 由??(文中未找到)负责cooling
- Huge page: 有`kmigrated`(sec 4.2.2)负责cooling

### Htmm flag

测试是否由htmm flag引起的sample被过滤问题

```diff
@@ -1006,12 +1020,17 @@ static int __update_pte_pginfo(struct vm_area_struct *vma, pmd_t *pmd,
                goto pte_unlock;
 
        pte_page = virt_to_page((unsigned long)pte);
-       if (!PageHtmm(pte_page))
+       if (!PageHtmm(pte_page)) {
+               ret = -EPERM;
                goto pte_unlock;
+       }
 
        pginfo = get_pginfo_from_pte(pte);
-       if (!pginfo)
+       if (!pginfo) {
+               ret = -ENOENT;
                goto pte_unlock;
+       }
```

```bash
sudo bpftrace -e 'kprobe:__update_pginfo { printf("__update_pginfo(vma=%p, addr=%p)", arg0, arg1) } kretprobe:__update_pginfo { printf(" = %d\n", retval) }' | tee /out/bpftrace.log
```

```shell
$ tail -n +2 out/0/bpftrace.log | awk '{print $4}' | sort -n | uniq -c
1950210 -1
     17 0
     62 1
   3509 2
```

结果发现返回值大部分都是`-1`即`EPERM`, 说明确实是flag没有被置位的原因.

 这个htmm flag只有在`mm->htmm_enabled`置位了的地址空间分配内存(页表项)时才会被设置(`pte_alloc_one()/__pte_alloc_pginfo()`).

检查每一个地址空间被创建时`htmm_enabled`的设置情况, 以及其对应的进程.

```diff
-void htmm_mm_init(struct mm_struct *mm)
+bool htmm_mm_init(struct mm_struct *mm)
 {
        struct mem_cgroup *memcg = get_mem_cgroup_from_mm(mm);
 
        if (!memcg || !memcg->htmm_enabled) {
-               mm->htmm_enabled = false;
-               return;
+               return mm->htmm_enabled = false;
        }
-       mm->htmm_enabled = true;
+       return mm->htmm_enabled = true;
 }
```

```bash
sudo bpftrace -e 'kprobe:mm_init { $p = (struct task_struct *)arg1; printf("mm_init(mm=%p, task=%p) pid=%d tgid=%d", arg0, arg1, $p->pid, $p->tgid)} kretprobe:htmm_mm_init {printf(" htmm_enabled=%d\n", retval)}' | tee /out/bpftrace.out
```

结果发现gups的进程并没有被开启htmm. 原因是我们采用了preload lib的方式为gups开启htmm, 但是这个时候`struct mm_struct`已经被创建了, 且其`htmm_enabled`已经被置空.

