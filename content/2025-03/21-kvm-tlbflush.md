+++
+++

### KVM

SDM中VMX章节总共描述了两种flush指令, 即`invept <eptp>`和`invvpid <vpid> <gva>`. 结合[之前的表格](12-ept-details)可以得到invvpid主要清除vpid tagged和vpid x EPTP tagged的TLB entries. 这种影响范围较小, 即只包括一个guest CPU相关的entries. 而invept则清除EPTP tagged和vpid x EPTP tagged的TLB entries. 这种影响范围则很大, 任何共用一个EPT的TLB entires都会被清楚. Guest的TLB flush包括两个指令: invpg和invpcid. invpcid和invvpid算是一对一映射的, 都只invalidate某个特定context的TLB entries. 在Guest中使用invpcid不会导致VM-exit. invpg虽然不是pcid aware, 但是其也不会触发VM-exit.

KVM的实现中不分青红皂白的直接使用invept, 导致误伤大量有效entries.

这里的原因实际上还是gVA reverse mapping缺失的问题. invvpid虽然可以单独清除某个gVA对应的TLB entries, 但是在host中做A-bit tracking只知道hVA/gPA. 而某个gPA却可能map到多个gVA. 同时host没法访问guest的rmap结构, 没法找出所有的gVA. 也就没法保证catch其他gVA的access. 为了保证正确性, 只能清除所有的TLB entries.

另外KVM的`remote_tlb_flush`实际上最终只会调用`invept`, 我们可以借此观察`invept`的调用次数.

| Where | What | `tlb_flush` | `remote_tlb_flush` | `tlb_local_flush_one` | `tlb_local_flush_all` | `tlb_remote_flush` | Elapsed  |
| ----- | ---- | ----------- | ------------------ | --------------------- | --------------------- | ------------------ | -------- |
| Guest | TPP  | 1240        | 0                  | 5,587,867             | 29,658                | 15,410,466         | 6:11.85  |
| Host  | TPP  | 89,361,453  | 21,220,056         | 5,889,040             | 22,998                | 1,205,535          | 32:37.02 |
|       | None | 26,402      | 0                  | 5669863               | 23846                 | 1179579            | 4:59.50  |
| Guest | PML  | 139,373     | 491                | 5606757               | 26806                 | 1180724            | 5:39.07  |
| Guest | Ours | 1888        | 0                  |                       |                       |                    |          |
|       |      |             |                    |                       |                       |                    |          |

```
sudo swapoff --all
sudo sysctl -w vm.overcommit_memory=1
echo 3 | sudo tee /proc/sys/vm/drop_caches
ulimit -n 65535
echo 3000000 | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_{min,max}_freq
echo 1 | sudo tee /proc/sys/vm/clear_all_vm_events                  
```

### Host

host only的情况下, 扫描A-bit, linux默认不立即flush TLB而是等到context switch. 见[这里](https://github.com/torvalds/linux/blob/0c3836482481200ead7b416ca80c68a29cfdaabd/arch/x86/mm/pgtable.c#L597).

```
sudo bpftrace -e 'tracepoint:tlb:tlb_flush { @[args.reason] = count() }'
Attaching 1 probe...
^C

@[4]: 42332
@[3]: 42333
@[0]: 393623
@[1]: 555818


```
