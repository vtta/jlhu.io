+++
+++

> The core insight behind our work is that existing hypervisor-based hotness tracking methods impose significant performance penalties.

我们rebuttal的主要观点是overhead. 可以通过motivation实验来很好的address. 目前SOTA的方法主要包括PML以及trapping. 另外包括reviewer A的MMU notifier建议. 我相信trapping是一票否决, 因为目前没有VM不采用硬件虚拟化. 目前需要demo的就包括MMU notifier以及PML. 我们只需要着重展现收sample环节的overhead即可. 因为后续环节即便将我们的优化设计放入SOTA, 他们也仍然会因为第一步的额外开销而整体性能更差.

DAMON扫描A-bit时会用到MMU notifier(`mmu_notifier_test_young`). 但是清除A-bit时却没有做TLB flush: 其使用了`mmu_notifier_clear_young`而不是`mmu_notifier_clear_flush_young`. 但是TPP扫描时(`folio_referenced`)却做了flush (`ptep_clear_flush_young_notify`). 目前我们认为TPP的做法是正确的. 因为如果不flush TLB, 新的访问不会设置内存中的A-bit. 根本原因在于地址翻译在看到TLB中有结果就返回了, 不会设置A-bit.  (这个说法已经被SDM验证, 相关信息更新到了[这里](11-mmu-notifier)和[这里](12-ept-details))

如果我们想采用TPP的方法, 需要为他加入手动触发扫描的方法. 目前TPP的触发依赖于kswapd, 即在内存不足的情况下才会扫描, 那么实际上我是否可以通过手动唤醒kswapd来触发TPP的扫描? 在此过程中我们记录下获得的A-bit信息以及总开销, 就能计算出平均per sample的tracking overhead. 总开销通过两次AB-testing完成, 即开与不开hotness tracking之间相差的时间.

结果发现不行, 因为kswapd扫描的时候只会根据需要释放多少内存去扫描, 如果这个数字是0则不会扫描.

综合DAMON和TPP我们决定纠正DAMON的tlbflush, 并使用DAMON抓取A-bit hotness. motivation实验中展示PEBS以及分别在host/guest中抓取A-bit hotness的平均开销.

另外还需要补充一下为什么只比较sampling的背景: host is not vm-aware.

#### 物理机模拟小VM环境准备

我们的虚拟机一般为4C16G. 其中FMEM占1/5, 即3276M. SMEM则为13108M. 则我们需要两种profile: a) LDRAM 3276M only; b) LDRAM 3276M + RDRAM 13108M. 来分别测试LPMEM以及RDRAM作为slower tier的情况.

通过kernel cmdline `maxcpus=4`只开启4个CPU, 然后通过cmdline只reserve一部份physical memory. 目前从dmesg中已知:

```
Node 0: 128G = [0, 2G) + [4G, 130G)
Node 1: 128G = [130G, 258G)
```

故node 0需要reserve [4G+(3276M-2G), 130G), node 1需要reserve [130G+13108M, 258G). 按照格式 `<size>$<start>`写为`memmap=127796M$5324M,117964M$146228M`

```
memmap=nn[KMG]$ss[KMG]
                [KNL,ACPI,EARLY] Mark specific memory as reserved.
                Region of memory to be reserved is from ss to ss+nn.
                Example: Exclude memory from 0x18690000-0x1869ffff
                         memmap=64K$0x18690000
                         or
                         memmap=0x10000$0x18690000
                Some bootloaders may need an escape character before '$',
                like Grub2, otherwise '$' and the following number
                will be eaten.
maxcpus=        [SMP,EARLY] Maximum number of processors that an SMP kernel
                will bring up during bootup.  maxcpus=n : n >= 0 limits
                the kernel to bring up 'n' processors. Surely after
                bootup you can bring up the other plugged cpu by executing
                "echo 1 > /sys/devices/system/cpu/cpuX/online". So maxcpus
                only takes effect during system bootup.
                While n=0 is a special case, it is equivalent to "nosmp",
                which also disables the IO APIC.
```

```
[    0.017089] ACPI: SRAT: Node 0 PXM 0 [mem 0x00000000-0x7fffffff]
[    0.017091] ACPI: SRAT: Node 0 PXM 0 [mem 0x100000000-0x207fffffff]
[    0.017092] ACPI: SRAT: Node 2 PXM 2 [mem 0x4080000000-0xbe7fffffff] non-volatile
[    0.017094] ACPI: SRAT: Node 1 PXM 1 [mem 0x2080000000-0x407fffffff]
[    0.017095] ACPI: SRAT: Node 3 PXM 3 [mem 0xbe80000000-0x13c7fffffff] non-volatile
[    0.017105] NUMA: Initialized distance table, cnt=4
[    0.017109] NUMA: Node 0 [mem 0x00000000-0x7fffffff] + [mem 0x100000000-0x207fffffff] -> [mem 0x00000000-0x207fffffff]
[    0.017120] NODE_DATA(0) allocated [mem 0x207ffd5000-0x207fffffff]
[    0.017144] NODE_DATA(1) allocated [mem 0x407ffd4000-0x407fffefff]
[    0.017332] Zone ranges:
[    0.017333]   DMA      [mem 0x0000000000001000-0x0000000000ffffff]
[    0.017335]   DMA32    [mem 0x0000000001000000-0x00000000ffffffff]
[    0.017337]   Normal   [mem 0x0000000100000000-0x000000407fffffff]
[    0.017339]   Device   empty
[    0.017340] Movable zone start for each node
[    0.017341] Early memory node ranges
[    0.017341]   node   0: [mem 0x0000000000001000-0x000000000003dfff]
[    0.017343]   node   0: [mem 0x000000000003f000-0x000000000009ffff]
[    0.017344]   node   0: [mem 0x0000000000100000-0x0000000068024fff]
[    0.017345]   node   0: [mem 0x000000006d3ff000-0x000000006f7fffff]
[    0.017346]   node   0: [mem 0x0000000100000000-0x000000207fffffff]
[    0.017354]   node   1: [mem 0x0000002080000000-0x000000407fffffff]
[    0.017362] Initmem setup node 0 [mem 0x0000000000001000-0x000000207fffffff]
[    0.017366] Initmem setup node 1 [mem 0x0000002080000000-0x000000407fffffff]
[    0.017386] Initmem setup node 2 as memoryless
[    0.017412] Initmem setup node 3 as memoryless
[    0.017414] On node 0, zone DMA: 1 pages in unavailable ranges
[    0.017417] On node 0, zone DMA: 1 pages in unavailable ranges
[    0.017451] On node 0, zone DMA: 96 pages in unavailable ranges
[    0.020897] On node 0, zone DMA32: 21466 pages in unavailable ranges
[    0.274007] On node 0, zone Normal: 2048 pages in unavailable ranges
```

开机测试发现OOM, 尝试只通过`mem=`限制总物理内存大小一样会引发OOM.

恢复正常启动, 查看刚开机的内存占用:

总共内存使用`MemTotal-MemAvailable`为8522M, 其中`VmallocUsed`以及`Cached`分别就占用了1156M和1231M. 然而占用内存最多的进程也不过几十兆.

另一种可能就是我们之前调大了PEBS buffer的大小. 仅限制CPU数量启动试一下. 结果总内存占用7597M, 两个node的占用内存分别为4309M和2434M. 而之前是3304M和4383M. 说明问题也不全在这里.

```
# sudo cat /proc/meminfo
MemTotal:       263714080 kB
MemFree:        255198208 kB
MemAvailable:   254986660 kB
Buffers:            9404 kB
Cached:          1260600 kB
SwapCached:            0 kB
Active:           615832 kB
Inactive:         868680 kB
Active(anon):     262988 kB
Inactive(anon):        0 kB
Active(file):     352844 kB
Inactive(file):   868680 kB
Unevictable:        9000 kB
Mlocked:               0 kB
SwapTotal:         65532 kB
SwapFree:          65532 kB
Zswap:                 0 kB
Zswapped:              0 kB
Dirty:               276 kB
Writeback:             0 kB
AnonPages:        219716 kB
Mapped:           217608 kB
Shmem:             48336 kB
KReclaimable:     123312 kB
Slab:             537532 kB
SReclaimable:     123312 kB
SUnreclaim:       414220 kB
KernelStack:       28464 kB
PageTables:         5248 kB
SecPageTables:     10428 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    131922572 kB
Committed_AS:     892944 kB
VmallocTotal:   34359738367 kB
VmallocUsed:     1184296 kB
VmallocChunk:          0 kB
Percpu:            89280 kB
HardwareCorrupted:     0 kB
AnonHugePages:    116736 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
FileHugePages:         0 kB
FilePmdMapped:         0 kB
Unaccepted:            0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
DirectMap4k:      196760 kB
DirectMap2M:     7835648 kB
DirectMap1G:    1319108608 kB
```

```
# sudo cat /proc/vmallocinfo | sort -n --key 2
# ...
0x000000004f2f103f-0x000000006572bf25 1052672 pci_iomap+0x72/0xc0 phys=0x0000217fffe00000 ioremap
0x00000000694c93a4-0x00000000f39dcffa 1052672 pci_iomap+0x72/0xc0 phys=0x0000203fffe00000 ioremap
0x00000000a482ac36-0x0000000055a58fde 1052672 alloc_large_system_hash+0x17b/0x280 pages=256 vmalloc N0=128 N1=128
0x0000000054bae7cf-0x0000000020f250e0 1245184 load_module+0x98d/0x1140 pages=303 vmalloc N0=303
0x00000000c237f308-0x00000000dcfee734 1376256 f2fs_build_segment_manager+0x3db/0x700 pages=335 vmalloc N0=335
0x0000000050a942eb-0x00000000ffa21e65 1576960 alloc_large_system_hash+0x17b/0x280 pages=384 vmalloc N0=192 N1=192
0x000000008f9c897a-0x00000000e3a5c6ab 1982464 acpi_os_map_iomem+0x11b/0x1c0 phys=0x000000006a643000 ioremap
0x00000000362b39a9-0x0000000032bb676f 2101248 pci_iomap+0x72/0xc0 phys=0x00000000fb400000 ioremap
0x000000004939cdc2-0x00000000db49b1f6 2101248 pci_iomap_range+0x78/0xc0 phys=0x000000009ae00000 ioremap
0x00000000665cb4b7-0x0000000050a942eb 2101248 alloc_large_system_hash+0x17b/0x280 pages=512 vmalloc N0=256 N1=256
0x00000000b40ca6d8-0x00000000d9eed70c 2101248 alloc_large_system_hash+0x17b/0x280 pages=512 vmalloc N0=256 N1=256
0x00000000bba2f5e7-0x00000000a482ac36 2101248 alloc_large_system_hash+0x17b/0x280 pages=512 vmalloc N0=256 N1=256
0x00000000ce3103c0-0x0000000021882154 2101248 alloc_large_system_hash+0x17b/0x280 pages=512 vmalloc N0=256 N1=256
0x00000000d9eed70c-0x000000004a761f5a 2101248 alloc_large_system_hash+0x17b/0x280 pages=512 vmalloc N0=256 N1=256
0x00000000ffa21e65-0x00000000ce3103c0 2101248 alloc_large_system_hash+0x17b/0x280 pages=512 vmalloc N0=256 N1=256
0x0000000078a06f28-0x000000009479c60e 2367488 memremap+0xdf/0x200 phys=0x0000000061380000 ioremap
0x000000002fd9e3f9-0x00000000665cb4b7 4198400 alloc_large_system_hash+0x17b/0x280 pages=1024 vmalloc vpages N0=512 N1=512
0x0000000056ba7a30-0x00000000bba2f5e7 4198400 alloc_large_system_hash+0x17b/0x280 pages=1024 vmalloc vpages N0=512 N1=512
0x0000000068c6d074-0x00000000ee190874 8392704 f2fs_build_segment_manager+0x374/0x700 pages=2048 vmalloc vpages N0=2048
0x00000000aa73e299-0x0000000066b2dd6d 8392704 bpf_jit_alloc_exec+0x1a/0x40 pages=2048 vmalloc vpages N0=1024 N1=1024
0x00000000274d5d0e-0x000000008271cdb2 9220096 drm_gem_shmem_vmap+0x1de/0x240 vmap
0x00000000a46db023-0x00000000646be28d 9220096 drm_fbdev_generic_helper_fb_probe+0xa6/0x1c0 pages=2250 vmalloc vpages N0=2250
0x0000000019bf3b68-0x0000000033036f13 12587008 f2fs_build_segment_manager+0x3db/0x700 pages=3072 vmalloc vpages N0=1536 N1=1536
0x00000000588d04bb-0x000000007e5cc219 14684160 devm_ioremap_wc+0x4e/0xc0 phys=0x000000009a000000 ioremap
0x00000000170054a9-0x0000000038c75916 16781312 stack_depot_init+0xaf/0x140 pages=4096 vmalloc vpages N0=2048 N1=2048
0x000000006150ded4-0x000000009b18425f 18874368 pcpu_get_vm_areas+0x0/0xfc0 vmalloc
0x000000008a054648-0x000000006150ded4 18874368 pcpu_get_vm_areas+0x0/0xfc0 vmalloc
0x00000000aee5953f-0x00000000b32d817d 18874368 pcpu_get_vm_areas+0x0/0xfc0 vmalloc
0x00000000b32d817d-0x000000007aa53c4e 18874368 pcpu_get_vm_areas+0x0/0xfc0 vmalloc
0x00000000dbefe978-0x00000000274d5d0e 33558528 f2fs_build_segment_manager+0x3db/0x700 pages=8192 vmalloc vpages N0=8192
0x000000007f9d3297-0x00000000b40ca6d8 67112960 alloc_large_system_hash+0x17b/0x280 pages=16384 vmalloc vpages N0=8192 N1=8192
0x00000000788a8cbb-0x000000005cbf11d7 134221824 alloc_large_system_hash+0x17b/0x280 pages=32768 vmalloc vpages N0=16384 N1=16384
0x000000002faaeb38-0x00000000c237f308 146804736 f2fs_build_segment_manager+0x374/0x700 pages=35840 vmalloc vpages N0=35840
0x000000007e81fcb6-0x000000006d71f041 268439552 pci_mmcfg_arch_map+0x33/0x80 phys=0x0000000080000000 ioremap
0x0000000071cbf760-0x0000000039cc3811 702550016 f2fs_build_segment_manager+0x3db/0x700 pages=171520 vmalloc vpages N0=171520
```

```
# sudo ps aux --sort=-%mem | head -10
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
jlhu        2180  1.6  0.0 535807536 120352 ?    Sl   13:05   0:02 /data/actions-runner/bin/Runner.Listener run --startuptype service
caddy       2517  0.1  0.0 1270316 46128 ?       Ssl  13:06   0:00 /usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
jlhu        2156  0.0  0.0 726640 39816 ?        Sl   13:05   0:00 ./externals/node20/bin/node ./bin/RunnerService.js
root        2284  0.2  0.0 1261060 39440 ?       Ssl  13:05   0:00 /usr/bin/cloudflared --no-autoupdate tunnel run
root        2106  0.1  0.0 368176 16388 ?        Ssl  13:05   0:00 /usr/bin/NetworkManager --no-daemon
systemd+    2089  0.0  0.0  20056 11780 ?        Ss   13:05   0:00 /usr/lib/systemd/systemd-resolved
root           1  1.2  0.0  21668 10540 ?        Ss   13:05   0:01 /usr/lib/systemd/systemd
jlhu        2356  0.0  0.0 406648  9512 ?        Sl   13:05   0:00 /usr/libexec/gvfsd-fuse /run/user/1000/gvfs -f
jlhu        2330  0.0  0.0  20068  9388 ?        Ss   13:05   0:00 /usr/lib/systemd/systemd --user
# sudo ps aux --sort=-rss | head -10
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
jlhu        2180  1.4  0.0 535807520 120352 ?    Sl   13:05   0:02 /data/actions-runner/bin/Runner.Listener run --startuptype service
caddy       2517  0.0  0.0 1270316 46128 ?       Ssl  13:06   0:00 /usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
jlhu        2156  0.0  0.0 726640 39816 ?        Sl   13:05   0:00 ./externals/node20/bin/node ./bin/RunnerService.js
root        2284  0.2  0.0 1261060 39440 ?       Ssl  13:05   0:00 /usr/bin/cloudflared --no-autoupdate tunnel run
root        2106  0.1  0.0 368176 16388 ?        Ssl  13:05   0:00 /usr/bin/NetworkManager --no-daemon
systemd+    2089  0.0  0.0  20056 11780 ?        Ss   13:05   0:00 /usr/lib/systemd/systemd-resolved
root           1  1.0  0.0  21668 10540 ?        Ss   13:05   0:01 /usr/lib/systemd/systemd
jlhu        2356  0.0  0.0 406648  9512 ?        Sl   13:05   0:00 /usr/libexec/gvfsd-fuse /run/user/1000/gvfs -f
jlhu        2330  0.0  0.0  20068  9388 ?        Ss   13:05   0:00 /usr/lib/systemd/systemd --user
```

```
# sudo numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107
node 0 size: 128536 MB
node 0 free: 125232 MB
node 1 cpus: 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 108 109 110 111 112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127 128 129 130 131 132 133 134 135 136 137 138 139 140 141 142 143
node 1 size: 128996 MB
node 1 free: 124613 MB
node distances:
node     0    1
   0:   10   20
   1:   20   10
# sudo numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3
node 0 size: 128593 MB
node 0 free: 124284 MB
node 1 cpus:
node 1 size: 128940 MB
node 1 free: 126506 MB
node distances:
node     0    1
   0:   10   20
   1:   20   10
```

实在不行就放大10倍, 用36C144G试一下. 采用1/4DRAM比例得到36G比上108G. 那这种情况下`memmap=90G$38G,128G$130G`

在我们内核下36C36G0G的setting下开机内存占用和不限制一样, 均为8G. 对测试的干扰算是更小一些.
