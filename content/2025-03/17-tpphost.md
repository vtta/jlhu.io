测试使用TPP作为host, 然后限制内存空间模拟一个大VM. 尝试36C36G0G.

切换到TPP后, 可用内存仅为18G. 重启后依旧.

检查是否是RNIC的问题, 卸载所有相关模组: `sudo lsmod | awk '/^(ib|mlx|rdma)/ {print $1}' | xargs sudo modprobe --remove --remove-holders` 后情况依旧.

尝试物理移除网卡设备看看. 并没有任何区别. 不限制可用CPU以及内存开机后依然是有~18G左右的占用.

现在还是怀疑是之前有过增大PEBS buffer的问题. 因为限制CPU数量后, 开机仍能看到offline的CPU, 怀疑CPU相关数据结构还是被kernel初始化了. 看log commit并未发现对`ds.c`的修改.

考虑之前6.10的kernel开机占用8G, 换成5.15占用18G, 应该着重排查kernel内部的占用情况.

尝试在kernel启动过程中开启ftrace.

```
ftrace=function
ftrace_boot_snapshot
trace_buf_size=16M
trace_event=kmem:mm_page_alloc
# only larger than 1MiB
trace_trigger="mm_page_alloc.stacktrace if order > 8"
trace_trigger="mm_page_alloc.hist:keys=common_stacktrace:vals=order if order > 8"
```

```shell
echo 0 | sudo tee /sys/kernel/debug/tracing/tracing_on
sudo cat /sys/kernel/debug/tracing/trace | tee trace.log
```

根据kernel文档中对于[cmdline](https://docs.kernel.org/admin-guide/kernel-parameters.html)以及[events](https://docs.kernel.org/trace/events.html)的描述, 使用`trace_trigger="mm_page_alloc.stacktrace if order > 8"`即可以找出较大块内存分配出现的stacktrace. 但可惜在TPP的v5.15并没有发现对`trace_trigger=`的支持. 但是在我们的kernel中有对应支持. 虽然我们的kernel启动时内存占用没有那么严重, 但是可以试试看, 可能出问题的地方是相似的. 

目前在我们6.10kernel中可以使用`trace_trigger`但不能使用hist trigger. 可能需要更新版本. stacktrace trigger也能用, 但是得手动统计出现次数. 目前看到出现次数最多的包括f2fs和watchdog的相关代码. 

watchdog代码会创建perf_event, 可能会受到PEBS buffer大小的影响. 检查代码发现, 我们并没有修改SOTA的PEBS buffer代码, 分配的buffer大小还是原本的2^4页. 而我们的kernel使用的是2^10页, 但总体内存占用却显著低于TPP. 可以排除PEBS带来的影响. 

使用nowatchdog选项关闭创建watchdog后内存占用并没有变化. 检查f2fs的问题. f2fs占用也就<1G.

最后屏蔽了nfit model, 发现内存恢复正常. 两个node的可用空间分别为127642/128549M和126958/128993M. 

这里应该是我们创建devdax的时候使用了系统内存作为来存memmap. 尝试改为dev.

```
$ sudo ndctl create-namespace -h
    -m, --mode <operation-mode>
                          specify a mode for the namespace, 'sector', 'fsdax', 'devdax' or 'raw'
    -M, --map <memmap-location>
                          specify 'mem' or 'dev' for the location of the memmap
```

对比使用mem和dev作为memmap时, 刚启动未上线pmem的内存占用情况:

```
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107
node 0 size: 128594 MB
node 0 free: 125522 MB
node 1 cpus: 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 108 109 110 111 112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127 128 129 130 131 132 133 134 135 136 137 138 139 140 141 142 143
node 1 size: 128940 MB
node 1 free: 124884 MB
node distances:
node     0    1
   0:   10   20
   1:   20   10
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107
node 0 size: 128594 MB
node 0 free: 127614 MB
node 1 cpus: 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 108 109 110 111 112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127 128 129 130 131 132 133 134 135 136 137 138 139 140 141 142 143
node 1 size: 128940 MB
node 1 free: 126916 MB
node distances:
node     0    1
   0:   10   20
   1:   20   10
```











```
        kernelcore=     [KNL,X86,PPC,EARLY]
                        Format: nn[KMGTPE] | nn% | "mirror"
                        This parameter specifies the amount of memory usable by
                        the kernel for non-movable allocations.  The requested
                        amount is spread evenly throughout all nodes in the
                        system as ZONE_NORMAL.  The remaining memory is used for
                        movable memory in its own zone, ZONE_MOVABLE.  In the
                        event, a node is too small to have both ZONE_NORMAL and
                        ZONE_MOVABLE, kernelcore memory will take priority and
                        other nodes will have a larger ZONE_MOVABLE.

                        ZONE_MOVABLE is used for the allocation of pages that
                        may be reclaimed or moved by the page migration
                        subsystem.  Note that allocations like PTEs-from-HighMem
                        still use the HighMem zone if it exists, and the Normal
                        zone if it does not.

                        It is possible to specify the exact amount of memory in
                        the form of "nn[KMGTPE]", a percentage of total system
                        memory in the form of "nn%", or "mirror".  If "mirror"
                        option is specified, mirrored (reliable) memory is used
                        for non-movable allocations and remaining memory is used
                        for Movable pages.  "nn[KMGTPE]", "nn%", and "mirror"
                        are exclusive, so you cannot specify multiple forms.
                        
        movablecore=    [KNL,X86,PPC,EARLY]
                        Format: nn[KMGTPE] | nn%
                        This parameter is the complement to kernelcore=, it
                        specifies the amount of memory used for migratable
                        allocations.  If both kernelcore and movablecore is
                        specified, then kernelcore will be at *least* the
                        specified value but may be more.  If movablecore on its
                        own is specified, the administrator must be careful
                        that the amount of memory usable for all allocations
                        is not too small.

        movable_node    [KNL,EARLY] Boot-time switch to make hotplugable memory
                        NUMA nodes to be movable. This means that the memory
                        of such nodes will be usable only for movable
                        allocations which rules out almost all kernel
                        allocations. Use with caution!
```
