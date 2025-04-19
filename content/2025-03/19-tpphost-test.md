+++
+++

### 最终结果

| TPP   | GUPS  | Throughput | Elapsed       |
| ----- | ----- | ---------- | ------------- |
| N/A   | Host  | 0.023545   | 284.800904083 |
| Host  | Host  | 0.061957   | 108.228113515 |
| Host  | Guest | 0.001556   | 4308.40182661 |
| Guest | Guest | 0.046232   | 145.041671566 |
| N/A   | Guest | 0.022034   | 304.330138055 |

```
data/virtiofsd --cache=never --socket-path=virtiofsd.data.socket --shared-dir=data


data/cloud-hypervisor --cpus boot=36 --memory size=144G,shared=on --kernel data/tpp/vmlinux.bin --cmdline "root=/dev/vda2 rw tsc=reliable console=ttyS0 mitigations=off cgroup_no_v1=all" --disk path=data/root0.img --net tap=ichb0,mac=2e:89:a8:e4:92:00 --fs tag=data,socket=virtiofsd.data.socket

sudo scripts/vm/tpp.py ssh -i scripts/vm/id_ed25519 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null clear@192.168.92.100 sudo /data/gups --thread=36 --update=7200000000 --len=135291469824 --granularity=8 --report 1000 hotset --hot=9663676416 --weight=9 --reverse
```

```
sudo sysctl -w kernel.kptr_restrict=0
sudo sysctl -w kernel.perf_event_paranoid=-1
sudo sysctl -w vm.overcommit_memory=1
sudo sysctl -w vm.compaction_proactiveness=0
sudo sysctl -w vm.extfrag_threshold=1000
echo 2 | sudo tee /sys/kernel/mm/ksm/run
echo 3 | sudo tee /proc/sys/vm/drop_caches
echo 1 | sudo tee /sys/kernel/tracing/options/funcgraph-retval
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
sudo swapoff -a
sudo mount -t virtiofs data /data
```

```
cat /proc/sys/kernel/numa_balancing
cat /proc/sys/vm/demote_scale_factor
cat /sys/kernel/mm/numa/demotion_enabled
sudo cat /sys/kernel/debug/kvm/remote_tlb_flush
sudo cat /sys/kernel/debug/kvm/tlb_flush
```
