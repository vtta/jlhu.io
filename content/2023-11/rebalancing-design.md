+++
title = "Rebalancing Design"
date = "2023-11-20"
+++

Gather all memory accesses and balloon info for every VM at a certain interval. Check if the desired proportion of accesses hit DRAM. 



## Why ballooning instead of hot-plugging?

Finer granularity. Hot-plugging requires page migration.

<details> <summary> Related code </summary>

```c
// mm/memory_hotplug.c:2147
static int try_offline_memory_block(struct memory_block *mem, void *arg)
{
    // ...
}

// mm/memory.c:229
static int memory_block_offline(struct memory_block *mem)
{
    // ...
    ret = offline_pages(start_pfn + nr_vmemmap_pages,
            nr_pages - nr_vmemmap_pages, mem->zone, mem->group);
    // ...
}

int __ref offline_pages(unsigned long start_pfn, unsigned long nr_pages,
			struct zone *zone, struct memory_group *group)
{
    // ...
    do_migrate_range(pfn, end_pfn);
    // ...
}
```
</details>



## What is the potential of vhost-balloon?

To verify if offloading balloon page freeing into host kernel can indeed cut down the context switch cost, we can compare the effect of existing vhost implementation of network devices.

Because these two devices share similar characteristics, i.e. no ack is actually needed when freeing a page via a balloon device or sending a packet via a network device.

Currently cloud-hypervisor offers no vhost-net implementation, we can perform this comparison in qemu.

| vhost-net | Host->VM | VM->Host |
| --------- | -------- | -------- |
| ON        | 27.4     | 21.8     |
| OFF       | 16.2     | 17.7     |
| Speedup   | 69%      | 23%      |

Because networking involve complex data processing which does not exist in ballooning, we expect these numbers to be the lower bound of vhost-balloon improvements.

<details> <summary> Raw data </summary>

<details><summary>vm launch script</summary>

```bash
#!/bin/bash

set -ex

VHOST_NET="${VHOST_NET:=on}"
TAP="${TAP:=ichb0}"
MAC="${MAC:=2e:89:a8:e4:92:00}"
ROOTFS="${ROOTFS:=/data/projects/bench/data/root0.img}"
KERNEL="${KERNEL:=/data/projects/bench/data/vmlinux.bin}"
CMDLINE=(
  console=ttyS0
  root=/dev/vda2
  rw
  rootfstype=ext4
  # console=hvc0
  module.sig_enforce=0
  mitigations=off
  cryptomgr.notests
  quiet
  init=/usr/lib/systemd/systemd-bootchart
  no_timer_check
  tsc=reliable
  noreplace-smp
  page_alloc.shuffle=0
  modprobe.blacklist=virtio_balloon
  transparent_hugepage=never
  loglevel=8
  ignore_loglevel
  log_buf_len=16M
)
CMDLINE="${CMDLINE[@]}"

VMARGS=(
  qemu-system-x86_64
  -enable-kvm
  -smp 4
  -m 8192
  -cpu host
  -machine type=q35

  -kernel "$KERNEL"
  -append "$CMDLINE"

  -drive if=none,file="$ROOTFS",id=rootfs
  -device virtio-blk-pci,drive=rootfs
  # virtio-blk-device makes qemu uses virtio-mmio instead of virtio-pci bus
  -netdev tap,id=tap0,script=no,downscript=no,ifname="$TAP",vhost="$VHOST_NET"
  -device virtio-net-pci,netdev=tap0,mac="$MAC"

  -display none
  -nographic
  # -serial tcp::4446,server,telnet
  # -monitor stdio
)

# echo "${VMARGS[@]}"
"${VMARGS[@]}"
```
</details>


<details> <summary> with vhost-net + Host->VM </summary>

Server
```bash
$ iperf3 --server --interval 1
warning: this system does not seem to support IPv6 - trying IPv4
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
Accepted connection from 192.168.92.1, port 38164
[  5] local 192.168.92.100 port 5201 connected to 192.168.92.1 port 38166
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  3.17 GBytes  27.3 Gbits/sec
[  5]   1.00-2.00   sec  3.19 GBytes  27.4 Gbits/sec
[  5]   2.00-3.00   sec  3.16 GBytes  27.1 Gbits/sec
[  5]   3.00-4.00   sec  3.15 GBytes  27.1 Gbits/sec
[  5]   4.00-5.00   sec  3.16 GBytes  27.1 Gbits/sec
[  5]   5.00-6.00   sec  3.15 GBytes  27.1 Gbits/sec
[  5]   6.00-7.00   sec  3.16 GBytes  27.1 Gbits/sec
[  5]   7.00-8.00   sec  3.15 GBytes  27.1 Gbits/sec
[  5]   8.00-9.00   sec  3.16 GBytes  27.2 Gbits/sec
[  5]   9.00-10.00  sec  3.18 GBytes  27.4 Gbits/sec
[  5]  10.00-11.00  sec  3.20 GBytes  27.5 Gbits/sec
[  5]  11.00-12.00  sec  3.19 GBytes  27.4 Gbits/sec
[  5]  12.00-13.00  sec  3.18 GBytes  27.3 Gbits/sec
[  5]  13.00-14.00  sec  3.19 GBytes  27.4 Gbits/sec
[  5]  14.00-15.00  sec  3.19 GBytes  27.4 Gbits/sec
[  5]  15.00-16.00  sec  3.20 GBytes  27.5 Gbits/sec
[  5]  16.00-17.00  sec  3.20 GBytes  27.5 Gbits/sec
[  5]  17.00-18.00  sec  3.20 GBytes  27.5 Gbits/sec
[  5]  18.00-19.00  sec  3.22 GBytes  27.6 Gbits/sec
[  5]  19.00-20.00  sec  3.17 GBytes  27.2 Gbits/sec
[  5]  20.00-21.00  sec  3.19 GBytes  27.4 Gbits/sec
[  5]  21.00-22.00  sec  3.18 GBytes  27.3 Gbits/sec
[  5]  22.00-23.00  sec  3.19 GBytes  27.4 Gbits/sec
[  5]  23.00-24.00  sec  3.18 GBytes  27.3 Gbits/sec
[  5]  24.00-25.00  sec  3.16 GBytes  27.2 Gbits/sec
[  5]  25.00-26.00  sec  3.17 GBytes  27.2 Gbits/sec
[  5]  26.00-27.00  sec  3.16 GBytes  27.1 Gbits/sec
[  5]  27.00-28.00  sec  3.16 GBytes  27.2 Gbits/sec
[  5]  28.00-29.00  sec  3.18 GBytes  27.3 Gbits/sec
[  5]  29.00-30.00  sec  3.20 GBytes  27.5 Gbits/sec
[  5]  30.00-31.00  sec  3.25 GBytes  27.9 Gbits/sec
[  5]  31.00-32.00  sec  3.21 GBytes  27.6 Gbits/sec
[  5]  32.00-33.00  sec  3.18 GBytes  27.3 Gbits/sec
[  5]  33.00-34.00  sec  3.19 GBytes  27.4 Gbits/sec
[  5]  34.00-35.00  sec  3.21 GBytes  27.6 Gbits/sec
[  5]  35.00-36.00  sec  3.19 GBytes  27.4 Gbits/sec
[  5]  36.00-37.00  sec  3.17 GBytes  27.3 Gbits/sec
[  5]  37.00-38.00  sec  3.18 GBytes  27.3 Gbits/sec
[  5]  38.00-39.00  sec  3.20 GBytes  27.4 Gbits/sec
[  5]  39.00-40.00  sec  3.21 GBytes  27.6 Gbits/sec
[  5]  40.00-41.00  sec  3.21 GBytes  27.6 Gbits/sec
[  5]  41.00-42.00  sec  3.20 GBytes  27.5 Gbits/sec
[  5]  42.00-43.00  sec  3.22 GBytes  27.6 Gbits/sec
[  5]  43.00-44.00  sec  3.18 GBytes  27.3 Gbits/sec
[  5]  44.00-45.00  sec  3.19 GBytes  27.4 Gbits/sec
[  5]  45.00-46.00  sec  3.22 GBytes  27.7 Gbits/sec
[  5]  46.00-47.00  sec  3.20 GBytes  27.5 Gbits/sec
[  5]  47.00-48.00  sec  3.24 GBytes  27.8 Gbits/sec
[  5]  48.00-49.00  sec  3.24 GBytes  27.9 Gbits/sec
[  5]  49.00-50.00  sec  3.23 GBytes  27.7 Gbits/sec
[  5]  50.00-51.00  sec  3.23 GBytes  27.8 Gbits/sec
[  5]  51.00-52.00  sec  3.21 GBytes  27.6 Gbits/sec
[  5]  52.00-53.00  sec  3.20 GBytes  27.5 Gbits/sec
[  5]  53.00-54.00  sec  3.23 GBytes  27.7 Gbits/sec
[  5]  54.00-55.00  sec  3.22 GBytes  27.6 Gbits/sec
[  5]  55.00-56.00  sec  3.21 GBytes  27.6 Gbits/sec
[  5]  56.00-57.00  sec  3.23 GBytes  27.7 Gbits/sec
[  5]  57.00-58.00  sec  3.21 GBytes  27.5 Gbits/sec
[  5]  58.00-59.00  sec  3.20 GBytes  27.5 Gbits/sec
[  5]  59.00-60.00  sec  3.24 GBytes  27.8 Gbits/sec
[  5]  60.00-60.00  sec  3.88 MBytes  26.4 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-60.00  sec   192 GBytes  27.4 Gbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201 (test #2)
-----------------------------------------------------------
^Ciperf3: interrupt - the server has terminated
```
Client
```bash
$ iperf3 --client 192.168.92.100 --interval 1 --time 60
Connecting to host 192.168.92.100, port 5201
[  5] local 192.168.92.1 port 38166 connected to 192.168.92.100 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  3.18 GBytes  27.3 Gbits/sec    0   3.11 MBytes
[  5]   1.00-2.00   sec  3.19 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]   2.00-3.00   sec  3.16 GBytes  27.1 Gbits/sec    0   3.11 MBytes
[  5]   3.00-4.00   sec  3.15 GBytes  27.1 Gbits/sec    0   3.11 MBytes
[  5]   4.00-5.00   sec  3.16 GBytes  27.1 Gbits/sec    0   3.11 MBytes
[  5]   5.00-6.00   sec  3.15 GBytes  27.1 Gbits/sec    0   3.11 MBytes
[  5]   6.00-7.00   sec  3.16 GBytes  27.1 Gbits/sec    0   3.11 MBytes
[  5]   7.00-8.00   sec  3.15 GBytes  27.1 Gbits/sec    0   3.11 MBytes
[  5]   8.00-9.00   sec  3.16 GBytes  27.1 Gbits/sec    0   3.11 MBytes
[  5]   9.00-10.00  sec  3.18 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]  10.00-11.00  sec  3.20 GBytes  27.5 Gbits/sec    0   3.11 MBytes
[  5]  11.00-12.00  sec  3.19 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]  12.00-13.00  sec  3.18 GBytes  27.3 Gbits/sec    0   3.11 MBytes
[  5]  13.00-14.00  sec  3.19 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]  14.00-15.00  sec  3.19 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]  15.00-16.00  sec  3.20 GBytes  27.5 Gbits/sec    0   3.11 MBytes
[  5]  16.00-17.00  sec  3.20 GBytes  27.5 Gbits/sec    0   3.11 MBytes
[  5]  17.00-18.00  sec  3.21 GBytes  27.5 Gbits/sec    0   3.11 MBytes
[  5]  18.00-19.00  sec  3.22 GBytes  27.6 Gbits/sec    0   3.11 MBytes
[  5]  19.00-20.00  sec  3.17 GBytes  27.2 Gbits/sec    0   3.11 MBytes
[  5]  20.00-21.00  sec  3.19 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]  21.00-22.00  sec  3.18 GBytes  27.3 Gbits/sec    0   3.11 MBytes
[  5]  22.00-23.00  sec  3.19 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]  23.00-24.00  sec  3.18 GBytes  27.3 Gbits/sec    0   3.11 MBytes
[  5]  24.00-25.00  sec  3.16 GBytes  27.2 Gbits/sec    0   3.11 MBytes
[  5]  25.00-26.00  sec  3.17 GBytes  27.2 Gbits/sec    0   3.11 MBytes
[  5]  26.00-27.00  sec  3.16 GBytes  27.1 Gbits/sec    0   3.11 MBytes
[  5]  27.00-28.00  sec  3.16 GBytes  27.2 Gbits/sec    0   3.11 MBytes
[  5]  28.00-29.00  sec  3.18 GBytes  27.3 Gbits/sec    0   3.11 MBytes
[  5]  29.00-30.00  sec  3.20 GBytes  27.5 Gbits/sec    0   3.11 MBytes
[  5]  30.00-31.00  sec  3.25 GBytes  27.9 Gbits/sec    0   3.11 MBytes
[  5]  31.00-32.00  sec  3.21 GBytes  27.6 Gbits/sec    0   3.11 MBytes
[  5]  32.00-33.00  sec  3.18 GBytes  27.3 Gbits/sec    0   3.11 MBytes
[  5]  33.00-34.00  sec  3.19 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]  34.00-35.00  sec  3.21 GBytes  27.6 Gbits/sec    0   3.11 MBytes
[  5]  35.00-36.00  sec  3.19 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]  36.00-37.00  sec  3.18 GBytes  27.3 Gbits/sec    0   3.11 MBytes
[  5]  37.00-38.00  sec  3.18 GBytes  27.3 Gbits/sec    0   3.11 MBytes
[  5]  38.00-39.00  sec  3.20 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]  39.00-40.00  sec  3.21 GBytes  27.6 Gbits/sec    0   3.11 MBytes
[  5]  40.00-41.00  sec  3.21 GBytes  27.6 Gbits/sec    0   3.11 MBytes
[  5]  41.00-42.00  sec  3.20 GBytes  27.5 Gbits/sec    0   3.11 MBytes
[  5]  42.00-43.00  sec  3.22 GBytes  27.6 Gbits/sec    0   3.11 MBytes
[  5]  43.00-44.00  sec  3.18 GBytes  27.3 Gbits/sec    0   3.11 MBytes
[  5]  44.00-45.00  sec  3.19 GBytes  27.4 Gbits/sec    0   3.11 MBytes
[  5]  45.00-46.00  sec  3.22 GBytes  27.7 Gbits/sec    0   3.11 MBytes
[  5]  46.00-47.00  sec  3.20 GBytes  27.5 Gbits/sec    0   3.11 MBytes
[  5]  47.00-48.00  sec  3.24 GBytes  27.8 Gbits/sec    0   3.11 MBytes
[  5]  48.00-49.00  sec  3.24 GBytes  27.9 Gbits/sec    0   3.11 MBytes
[  5]  49.00-50.00  sec  3.23 GBytes  27.7 Gbits/sec    0   3.11 MBytes
[  5]  50.00-51.00  sec  3.23 GBytes  27.8 Gbits/sec    0   3.11 MBytes
[  5]  51.00-52.00  sec  3.21 GBytes  27.6 Gbits/sec    0   3.11 MBytes
[  5]  52.00-53.00  sec  3.20 GBytes  27.5 Gbits/sec    0   3.11 MBytes
[  5]  53.00-54.00  sec  3.23 GBytes  27.7 Gbits/sec    0   3.11 MBytes
[  5]  54.00-55.00  sec  3.22 GBytes  27.6 Gbits/sec    0   3.11 MBytes
[  5]  55.00-56.00  sec  3.21 GBytes  27.6 Gbits/sec    0   3.11 MBytes
[  5]  56.00-57.00  sec  3.23 GBytes  27.7 Gbits/sec    0   3.11 MBytes
[  5]  57.00-58.00  sec  3.21 GBytes  27.5 Gbits/sec    0   3.11 MBytes
[  5]  58.00-59.00  sec  3.20 GBytes  27.5 Gbits/sec    0   3.11 MBytes
[  5]  59.00-60.00  sec  3.24 GBytes  27.8 Gbits/sec    0   3.11 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec   192 GBytes  27.4 Gbits/sec    0             sender
[  5]   0.00-60.00  sec   192 GBytes  27.4 Gbits/sec                  receiver

iperf Done.
```
</details>



<details> <summary> with vhost-net + VM->Host </summary>

Server
```bash
$ iperf3 --server --interval 1
warning: this system does not seem to support IPv6 - trying IPv4
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
Accepted connection from 192.168.92.100, port 57634
[  5] local 192.168.92.1 port 5201 connected to 192.168.92.100 port 57642
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  2.57 GBytes  22.0 Gbits/sec
[  5]   1.00-2.00   sec  2.57 GBytes  22.1 Gbits/sec
[  5]   2.00-3.00   sec  2.54 GBytes  21.8 Gbits/sec
[  5]   3.00-4.00   sec  2.53 GBytes  21.8 Gbits/sec
[  5]   4.00-5.00   sec  2.55 GBytes  21.9 Gbits/sec
[  5]   5.00-6.00   sec  2.54 GBytes  21.8 Gbits/sec
[  5]   6.00-7.00   sec  2.51 GBytes  21.5 Gbits/sec
[  5]   7.00-8.00   sec  2.54 GBytes  21.9 Gbits/sec
[  5]   8.00-9.00   sec  2.54 GBytes  21.8 Gbits/sec
[  5]   9.00-10.00  sec  2.53 GBytes  21.7 Gbits/sec
[  5]  10.00-11.00  sec  2.51 GBytes  21.6 Gbits/sec
[  5]  11.00-12.00  sec  2.51 GBytes  21.6 Gbits/sec
[  5]  12.00-13.00  sec  2.54 GBytes  21.8 Gbits/sec
[  5]  13.00-14.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  14.00-15.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  15.00-16.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  16.00-17.00  sec  2.52 GBytes  21.7 Gbits/sec
[  5]  17.00-18.00  sec  2.53 GBytes  21.7 Gbits/sec
[  5]  18.00-19.00  sec  2.54 GBytes  21.8 Gbits/sec
[  5]  19.00-20.00  sec  2.54 GBytes  21.9 Gbits/sec
[  5]  20.00-21.00  sec  2.56 GBytes  22.0 Gbits/sec
[  5]  21.00-22.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  22.00-23.00  sec  2.45 GBytes  21.1 Gbits/sec
[  5]  23.00-24.00  sec  2.52 GBytes  21.6 Gbits/sec
[  5]  24.00-25.00  sec  2.54 GBytes  21.8 Gbits/sec
[  5]  25.00-26.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  26.00-27.00  sec  2.54 GBytes  21.9 Gbits/sec
[  5]  27.00-28.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  28.00-29.00  sec  2.53 GBytes  21.8 Gbits/sec
[  5]  29.00-30.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  30.00-31.00  sec  2.56 GBytes  22.0 Gbits/sec
[  5]  31.00-32.00  sec  2.56 GBytes  22.0 Gbits/sec
[  5]  32.00-33.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  33.00-34.00  sec  2.52 GBytes  21.7 Gbits/sec
[  5]  34.00-35.00  sec  2.52 GBytes  21.6 Gbits/sec
[  5]  35.00-36.00  sec  2.54 GBytes  21.8 Gbits/sec
[  5]  36.00-37.00  sec  2.54 GBytes  21.8 Gbits/sec
[  5]  37.00-38.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  38.00-39.00  sec  2.54 GBytes  21.8 Gbits/sec
[  5]  39.00-40.00  sec  2.50 GBytes  21.4 Gbits/sec
[  5]  40.00-41.00  sec  2.54 GBytes  21.8 Gbits/sec
[  5]  41.00-42.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  42.00-43.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  43.00-44.00  sec  2.56 GBytes  22.0 Gbits/sec
[  5]  44.00-45.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  45.00-46.00  sec  2.56 GBytes  22.0 Gbits/sec
[  5]  46.00-47.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  47.00-48.00  sec  2.56 GBytes  22.0 Gbits/sec
[  5]  48.00-49.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  49.00-50.00  sec  2.56 GBytes  22.0 Gbits/sec
[  5]  50.00-51.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  51.00-52.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  52.00-53.00  sec  2.57 GBytes  22.0 Gbits/sec
[  5]  53.00-54.00  sec  2.57 GBytes  22.1 Gbits/sec
[  5]  54.00-55.00  sec  2.56 GBytes  22.0 Gbits/sec
[  5]  55.00-56.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  56.00-57.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  57.00-58.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  58.00-59.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  59.00-60.00  sec  2.55 GBytes  21.9 Gbits/sec
[  5]  60.00-60.00  sec   512 KBytes  24.5 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-60.00  sec   153 GBytes  21.8 Gbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201 (test #2)
-----------------------------------------------------------
^Ciperf3: interrupt - the server has terminated
```
Client
```bash
$ iperf3 --client 192.168.92.1 --interval 1 --time 60
Connecting to host 192.168.92.1, port 5201
[  5] local 192.168.92.100 port 57642 connected to 192.168.92.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  2.57 GBytes  22.1 Gbits/sec    0   1.79 MBytes
[  5]   1.00-2.00   sec  2.57 GBytes  22.1 Gbits/sec    0   1.79 MBytes
[  5]   2.00-3.00   sec  2.54 GBytes  21.8 Gbits/sec    0   1.90 MBytes
[  5]   3.00-4.00   sec  2.53 GBytes  21.8 Gbits/sec    0   2.02 MBytes
[  5]   4.00-5.00   sec  2.55 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]   5.00-6.00   sec  2.54 GBytes  21.8 Gbits/sec    0   2.02 MBytes
[  5]   6.00-7.00   sec  2.51 GBytes  21.6 Gbits/sec    0   2.02 MBytes
[  5]   7.00-8.00   sec  2.54 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]   8.00-9.00   sec  2.54 GBytes  21.8 Gbits/sec    0   2.02 MBytes
[  5]   9.00-10.00  sec  2.53 GBytes  21.7 Gbits/sec    0   2.02 MBytes
[  5]  10.00-11.00  sec  2.51 GBytes  21.6 Gbits/sec    0   2.02 MBytes
[  5]  11.00-12.00  sec  2.51 GBytes  21.6 Gbits/sec    0   2.02 MBytes
[  5]  12.00-13.00  sec  2.54 GBytes  21.8 Gbits/sec    0   2.02 MBytes
[  5]  13.00-14.00  sec  2.55 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]  14.00-15.00  sec  2.55 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]  15.00-16.00  sec  2.55 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]  16.00-17.00  sec  2.52 GBytes  21.7 Gbits/sec    0   2.02 MBytes
[  5]  17.00-18.00  sec  2.53 GBytes  21.8 Gbits/sec    0   2.02 MBytes
[  5]  18.00-19.00  sec  2.54 GBytes  21.8 Gbits/sec    0   2.02 MBytes
[  5]  19.00-20.00  sec  2.55 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]  20.00-21.00  sec  2.56 GBytes  22.0 Gbits/sec    0   2.02 MBytes
[  5]  21.00-22.00  sec  2.55 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]  22.00-23.00  sec  2.45 GBytes  21.1 Gbits/sec    0   2.02 MBytes
[  5]  23.00-24.00  sec  2.52 GBytes  21.6 Gbits/sec    0   2.02 MBytes
[  5]  24.00-25.00  sec  2.54 GBytes  21.8 Gbits/sec    0   2.02 MBytes
[  5]  25.00-26.00  sec  2.55 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]  26.00-27.00  sec  2.54 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]  27.00-28.00  sec  2.55 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]  28.00-29.00  sec  2.53 GBytes  21.8 Gbits/sec    0   2.02 MBytes
[  5]  29.00-30.00  sec  2.55 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]  30.00-31.00  sec  2.56 GBytes  22.0 Gbits/sec    0   2.02 MBytes
[  5]  31.00-32.00  sec  2.56 GBytes  22.0 Gbits/sec    0   2.02 MBytes
[  5]  32.00-33.00  sec  2.54 GBytes  21.9 Gbits/sec    0   2.02 MBytes
[  5]  33.00-34.00  sec  2.52 GBytes  21.7 Gbits/sec    0   2.02 MBytes
[  5]  34.00-35.00  sec  2.52 GBytes  21.6 Gbits/sec    0   2.02 MBytes
[  5]  35.00-36.00  sec  2.54 GBytes  21.8 Gbits/sec    0   2.02 MBytes
[  5]  36.00-37.00  sec  2.54 GBytes  21.8 Gbits/sec    0   3.33 MBytes
[  5]  37.00-38.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  38.00-39.00  sec  2.54 GBytes  21.8 Gbits/sec    0   3.33 MBytes
[  5]  39.00-40.00  sec  2.50 GBytes  21.5 Gbits/sec    0   3.33 MBytes
[  5]  40.00-41.00  sec  2.54 GBytes  21.8 Gbits/sec    0   3.33 MBytes
[  5]  41.00-42.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  42.00-43.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  43.00-44.00  sec  2.55 GBytes  22.0 Gbits/sec    0   3.33 MBytes
[  5]  44.00-45.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  45.00-46.00  sec  2.55 GBytes  22.0 Gbits/sec    0   3.33 MBytes
[  5]  46.00-47.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  47.00-48.00  sec  2.56 GBytes  22.0 Gbits/sec    0   3.33 MBytes
[  5]  48.00-49.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  49.00-50.00  sec  2.56 GBytes  22.0 Gbits/sec    0   3.33 MBytes
[  5]  50.00-51.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  51.00-52.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  52.00-53.00  sec  2.57 GBytes  22.0 Gbits/sec    0   3.33 MBytes
[  5]  53.00-54.00  sec  2.57 GBytes  22.1 Gbits/sec    0   3.33 MBytes
[  5]  54.00-55.00  sec  2.56 GBytes  22.0 Gbits/sec    0   3.33 MBytes
[  5]  55.00-56.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  56.00-57.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  57.00-58.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  58.00-59.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
[  5]  59.00-60.00  sec  2.55 GBytes  21.9 Gbits/sec    0   3.33 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec   153 GBytes  21.8 Gbits/sec    0             sender
[  5]   0.00-60.00  sec   153 GBytes  21.8 Gbits/sec                  receiver

iperf Done.
```
</details>


<details> <summary> with vhost-net + VM0->VM1 </summary>

Server
```bash
 $ iperf3 --server --interval 1
warning: this system does not seem to support IPv6 - trying IPv4
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
Accepted connection from 192.168.92.100, port 48982
[  5] local 192.168.92.101 port 5201 connected to 192.168.92.100 port 48984
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  2.08 GBytes  17.8 Gbits/sec
[  5]   1.00-2.00   sec  2.08 GBytes  17.9 Gbits/sec
[  5]   2.00-3.00   sec  2.09 GBytes  17.9 Gbits/sec
[  5]   3.00-4.00   sec  2.09 GBytes  17.9 Gbits/sec
[  5]   4.00-5.00   sec  2.06 GBytes  17.7 Gbits/sec
[  5]   5.00-6.00   sec  1.98 GBytes  17.0 Gbits/sec
[  5]   6.00-7.00   sec  2.03 GBytes  17.4 Gbits/sec
[  5]   7.00-8.00   sec  2.05 GBytes  17.6 Gbits/sec
[  5]   8.00-9.00   sec  2.05 GBytes  17.6 Gbits/sec
[  5]   9.00-10.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  10.00-11.00  sec  2.03 GBytes  17.4 Gbits/sec
[  5]  11.00-12.00  sec  2.03 GBytes  17.5 Gbits/sec
[  5]  12.00-13.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  13.00-14.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  14.00-15.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  15.00-16.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  16.00-17.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  17.00-18.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  18.00-19.00  sec  2.04 GBytes  17.6 Gbits/sec
[  5]  19.00-20.00  sec  2.04 GBytes  17.6 Gbits/sec
[  5]  20.00-21.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  21.00-22.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  22.00-23.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  23.00-24.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  24.00-25.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  25.00-26.00  sec  2.04 GBytes  17.6 Gbits/sec
[  5]  26.00-27.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  27.00-28.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  28.00-29.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  29.00-30.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  30.00-31.00  sec  2.09 GBytes  17.9 Gbits/sec
[  5]  31.00-32.00  sec  2.11 GBytes  18.1 Gbits/sec
[  5]  32.00-33.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  33.00-34.00  sec  2.09 GBytes  18.0 Gbits/sec
[  5]  34.00-35.00  sec  2.17 GBytes  18.6 Gbits/sec
[  5]  35.00-36.00  sec  2.17 GBytes  18.6 Gbits/sec
[  5]  36.00-37.00  sec  2.17 GBytes  18.6 Gbits/sec
[  5]  37.00-38.00  sec  2.16 GBytes  18.6 Gbits/sec
[  5]  38.00-39.00  sec  2.16 GBytes  18.6 Gbits/sec
[  5]  39.00-40.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  40.00-41.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  41.00-42.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  42.00-43.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  43.00-44.00  sec  2.04 GBytes  17.6 Gbits/sec
[  5]  44.00-45.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  45.00-46.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  46.00-47.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  47.00-48.00  sec  2.17 GBytes  18.6 Gbits/sec
[  5]  48.00-49.00  sec  2.16 GBytes  18.5 Gbits/sec
[  5]  49.00-50.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  50.00-51.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  51.00-52.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  52.00-53.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  53.00-54.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  54.00-55.00  sec  2.16 GBytes  18.5 Gbits/sec
[  5]  55.00-56.00  sec  2.17 GBytes  18.7 Gbits/sec
[  5]  56.00-57.00  sec  2.18 GBytes  18.8 Gbits/sec
[  5]  57.00-58.00  sec  2.17 GBytes  18.7 Gbits/sec
[  5]  58.00-59.00  sec  2.16 GBytes  18.6 Gbits/sec
[  5]  59.00-60.00  sec  2.02 GBytes  17.4 Gbits/sec
[  5]  60.00-60.00  sec  1.25 MBytes  18.4 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-60.00  sec   124 GBytes  17.8 Gbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201 (test #2)
-----------------------------------------------------------
^Ciperf3: interrupt - the server has terminated
```
Client
```bash
 $ iperf3 --client 192.168.92.101 --interval 1 --time 60
Connecting to host 192.168.92.101, port 5201
[  5] local 192.168.92.100 port 48984 connected to 192.168.92.101 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  2.08 GBytes  17.9 Gbits/sec    0   2.51 MBytes
[  5]   1.00-2.00   sec  2.08 GBytes  17.9 Gbits/sec    0   2.51 MBytes
[  5]   2.00-3.00   sec  2.09 GBytes  17.9 Gbits/sec    0   2.51 MBytes
[  5]   3.00-4.00   sec  2.09 GBytes  17.9 Gbits/sec    0   2.51 MBytes
[  5]   4.00-5.00   sec  2.06 GBytes  17.7 Gbits/sec    0   2.78 MBytes
[  5]   5.00-6.00   sec  1.98 GBytes  17.0 Gbits/sec    0   2.95 MBytes
[  5]   6.00-7.00   sec  2.03 GBytes  17.4 Gbits/sec    0   2.95 MBytes
[  5]   7.00-8.00   sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]   8.00-9.00   sec  2.04 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]   9.00-10.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  10.00-11.00  sec  2.03 GBytes  17.4 Gbits/sec    0   3.12 MBytes
[  5]  11.00-12.00  sec  2.03 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  12.00-13.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  13.00-14.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  14.00-15.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  15.00-16.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  16.00-17.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  17.00-18.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  18.00-19.00  sec  2.04 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  19.00-20.00  sec  2.04 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  20.00-21.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  21.00-22.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  22.00-23.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  23.00-24.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  24.00-25.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  25.00-26.00  sec  2.04 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  26.00-27.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  27.00-28.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  28.00-29.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  29.00-30.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.12 MBytes
[  5]  30.00-31.00  sec  2.09 GBytes  18.0 Gbits/sec    0   3.12 MBytes
[  5]  31.00-32.00  sec  2.11 GBytes  18.1 Gbits/sec    0   3.12 MBytes
[  5]  32.00-33.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.12 MBytes
[  5]  33.00-34.00  sec  2.09 GBytes  18.0 Gbits/sec    0   3.12 MBytes
[  5]  34.00-35.00  sec  2.17 GBytes  18.6 Gbits/sec    0   3.12 MBytes
[  5]  35.00-36.00  sec  2.17 GBytes  18.6 Gbits/sec    0   3.12 MBytes
[  5]  36.00-37.00  sec  2.17 GBytes  18.6 Gbits/sec    0   3.12 MBytes
[  5]  37.00-38.00  sec  2.16 GBytes  18.6 Gbits/sec    0   3.12 MBytes
[  5]  38.00-39.00  sec  2.16 GBytes  18.6 Gbits/sec    0   3.12 MBytes
[  5]  39.00-40.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  40.00-41.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  41.00-42.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  42.00-43.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.12 MBytes
[  5]  43.00-44.00  sec  2.04 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  44.00-45.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  45.00-46.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  46.00-47.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.12 MBytes
[  5]  47.00-48.00  sec  2.17 GBytes  18.6 Gbits/sec    0   3.12 MBytes
[  5]  48.00-49.00  sec  2.16 GBytes  18.5 Gbits/sec    0   3.12 MBytes
[  5]  49.00-50.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  50.00-51.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  51.00-52.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  52.00-53.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.12 MBytes
[  5]  53.00-54.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.12 MBytes
[  5]  54.00-55.00  sec  2.16 GBytes  18.5 Gbits/sec    0   3.12 MBytes
[  5]  55.00-56.00  sec  2.17 GBytes  18.7 Gbits/sec    0   3.12 MBytes
[  5]  56.00-57.00  sec  2.18 GBytes  18.8 Gbits/sec    0   3.12 MBytes
[  5]  57.00-58.00  sec  2.17 GBytes  18.7 Gbits/sec    0   3.12 MBytes
[  5]  58.00-59.00  sec  2.16 GBytes  18.6 Gbits/sec    0   3.12 MBytes
[  5]  59.00-60.00  sec  2.03 GBytes  17.4 Gbits/sec    0   3.12 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec   124 GBytes  17.8 Gbits/sec    0             sender
[  5]   0.00-60.00  sec   124 GBytes  17.8 Gbits/sec                  receiver

iperf Done.
```
</details>


<details> <summary> with vhost-net + VM1->VM0 </summary>

Server
```bash
$ iperf3 --server --interval 1
warning: this system does not seem to support IPv6 - trying IPv4
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
Accepted connection from 192.168.92.101, port 54936
[  5] local 192.168.92.100 port 5201 connected to 192.168.92.101 port 54948
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec   782 MBytes  6.55 Gbits/sec
[  5]   1.00-2.00   sec   779 MBytes  6.53 Gbits/sec
[  5]   2.00-3.00   sec   803 MBytes  6.73 Gbits/sec
[  5]   3.00-4.00   sec   796 MBytes  6.68 Gbits/sec
[  5]   4.00-5.00   sec   792 MBytes  6.65 Gbits/sec
[  5]   5.00-6.00   sec   788 MBytes  6.61 Gbits/sec
[  5]   6.00-7.00   sec   768 MBytes  6.44 Gbits/sec
[  5]   7.00-8.00   sec   778 MBytes  6.53 Gbits/sec
[  5]   8.00-9.00   sec   740 MBytes  6.21 Gbits/sec
[  5]   9.00-10.00  sec   740 MBytes  6.21 Gbits/sec
[  5]  10.00-11.00  sec   743 MBytes  6.23 Gbits/sec
[  5]  11.00-12.00  sec   746 MBytes  6.26 Gbits/sec
[  5]  12.00-13.00  sec   743 MBytes  6.23 Gbits/sec
[  5]  13.00-14.00  sec   748 MBytes  6.27 Gbits/sec
[  5]  14.00-15.00  sec   738 MBytes  6.19 Gbits/sec
[  5]  15.00-16.00  sec   751 MBytes  6.30 Gbits/sec
[  5]  16.00-17.00  sec   706 MBytes  5.93 Gbits/sec
[  5]  17.00-18.00  sec   719 MBytes  6.03 Gbits/sec
[  5]  18.00-19.00  sec   729 MBytes  6.12 Gbits/sec
[  5]  19.00-20.00  sec   725 MBytes  6.08 Gbits/sec
[  5]  20.00-21.00  sec   721 MBytes  6.05 Gbits/sec
[  5]  21.00-22.00  sec   725 MBytes  6.08 Gbits/sec
[  5]  22.00-23.00  sec   720 MBytes  6.04 Gbits/sec
[  5]  23.00-24.00  sec   714 MBytes  5.99 Gbits/sec
[  5]  24.00-25.00  sec   688 MBytes  5.77 Gbits/sec
[  5]  25.00-26.00  sec   714 MBytes  5.99 Gbits/sec
[  5]  26.00-27.00  sec   709 MBytes  5.95 Gbits/sec
[  5]  27.00-28.00  sec   728 MBytes  6.11 Gbits/sec
[  5]  28.00-29.00  sec   708 MBytes  5.94 Gbits/sec
[  5]  29.00-30.00  sec   711 MBytes  5.96 Gbits/sec
[  5]  30.00-31.00  sec   745 MBytes  6.25 Gbits/sec
[  5]  31.00-32.00  sec   755 MBytes  6.33 Gbits/sec
[  5]  32.00-33.00  sec   756 MBytes  6.34 Gbits/sec
[  5]  33.00-34.00  sec   764 MBytes  6.41 Gbits/sec
[  5]  34.00-35.00  sec   767 MBytes  6.43 Gbits/sec
[  5]  35.00-36.00  sec   765 MBytes  6.42 Gbits/sec
[  5]  36.00-37.00  sec   745 MBytes  6.25 Gbits/sec
[  5]  37.00-38.00  sec   771 MBytes  6.46 Gbits/sec
[  5]  38.00-39.00  sec   778 MBytes  6.53 Gbits/sec
[  5]  39.00-40.00  sec   779 MBytes  6.53 Gbits/sec
[  5]  40.00-41.00  sec   777 MBytes  6.52 Gbits/sec
[  5]  41.00-42.00  sec   783 MBytes  6.57 Gbits/sec
[  5]  42.00-43.00  sec   790 MBytes  6.63 Gbits/sec
[  5]  43.00-44.00  sec   788 MBytes  6.61 Gbits/sec
[  5]  44.00-45.00  sec   813 MBytes  6.82 Gbits/sec
[  5]  45.00-46.00  sec   818 MBytes  6.87 Gbits/sec
[  5]  46.00-47.00  sec   772 MBytes  6.47 Gbits/sec
[  5]  47.00-48.00  sec   773 MBytes  6.49 Gbits/sec
[  5]  48.00-49.00  sec   770 MBytes  6.46 Gbits/sec
[  5]  49.00-50.00  sec   772 MBytes  6.47 Gbits/sec
[  5]  50.00-51.00  sec   770 MBytes  6.46 Gbits/sec
[  5]  51.00-52.00  sec   770 MBytes  6.46 Gbits/sec
[  5]  52.00-53.00  sec   768 MBytes  6.45 Gbits/sec
[  5]  53.00-54.00  sec   768 MBytes  6.44 Gbits/sec
[  5]  54.00-55.00  sec   772 MBytes  6.48 Gbits/sec
[  5]  55.00-56.00  sec   778 MBytes  6.53 Gbits/sec
[  5]  56.00-57.00  sec   749 MBytes  6.28 Gbits/sec
[  5]  57.00-58.00  sec   769 MBytes  6.45 Gbits/sec
[  5]  58.00-59.00  sec   766 MBytes  6.42 Gbits/sec
[  5]  59.00-60.00  sec   768 MBytes  6.44 Gbits/sec
[  5]  60.00-60.00  sec   640 KBytes  5.98 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-60.00  sec  44.3 GBytes  6.35 Gbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201 (test #2)
-----------------------------------------------------------
^Ciperf3: interrupt - the server has terminated
```
Client
```bash
$ iperf3 --client 192.168.92.100 --interval 1 --time 60
Connecting to host 192.168.92.100, port 5201
[  5] local 192.168.92.101 port 54948 connected to 192.168.92.100 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   785 MBytes  6.58 Gbits/sec    0   1.80 MBytes
[  5]   1.00-2.00   sec   779 MBytes  6.53 Gbits/sec    0   2.13 MBytes
[  5]   2.00-3.00   sec   802 MBytes  6.73 Gbits/sec    0   2.13 MBytes
[  5]   3.00-4.00   sec   796 MBytes  6.68 Gbits/sec    0   2.32 MBytes
[  5]   4.00-5.00   sec   792 MBytes  6.65 Gbits/sec    0   2.59 MBytes
[  5]   5.00-6.00   sec   788 MBytes  6.61 Gbits/sec    0   2.59 MBytes
[  5]   6.00-7.00   sec   769 MBytes  6.45 Gbits/sec    0   2.73 MBytes
[  5]   7.00-8.00   sec   778 MBytes  6.52 Gbits/sec    0   2.73 MBytes
[  5]   8.00-9.00   sec   740 MBytes  6.21 Gbits/sec    0   2.73 MBytes
[  5]   9.00-10.00  sec   740 MBytes  6.21 Gbits/sec    0   2.73 MBytes
[  5]  10.00-11.00  sec   742 MBytes  6.23 Gbits/sec    0   2.73 MBytes
[  5]  11.00-12.00  sec   746 MBytes  6.26 Gbits/sec    0   2.73 MBytes
[  5]  12.00-13.00  sec   744 MBytes  6.24 Gbits/sec    0   2.73 MBytes
[  5]  13.00-14.00  sec   748 MBytes  6.27 Gbits/sec    0   2.73 MBytes
[  5]  14.00-15.00  sec   738 MBytes  6.19 Gbits/sec    0   2.73 MBytes
[  5]  15.00-16.00  sec   751 MBytes  6.30 Gbits/sec    0   2.73 MBytes
[  5]  16.00-17.00  sec   706 MBytes  5.92 Gbits/sec    0   2.73 MBytes
[  5]  17.00-18.00  sec   719 MBytes  6.03 Gbits/sec    0   2.73 MBytes
[  5]  18.00-19.00  sec   730 MBytes  6.12 Gbits/sec    0   2.73 MBytes
[  5]  19.00-20.00  sec   724 MBytes  6.07 Gbits/sec    0   2.73 MBytes
[  5]  20.00-21.00  sec   721 MBytes  6.05 Gbits/sec    0   2.73 MBytes
[  5]  21.00-22.00  sec   725 MBytes  6.08 Gbits/sec    0   2.73 MBytes
[  5]  22.00-23.00  sec   721 MBytes  6.05 Gbits/sec    0   2.73 MBytes
[  5]  23.00-24.00  sec   714 MBytes  5.99 Gbits/sec    0   2.73 MBytes
[  5]  24.00-25.00  sec   688 MBytes  5.77 Gbits/sec    0   2.73 MBytes
[  5]  25.00-26.00  sec   714 MBytes  5.99 Gbits/sec    0   2.73 MBytes
[  5]  26.00-27.00  sec   710 MBytes  5.96 Gbits/sec    0   2.73 MBytes
[  5]  27.00-28.00  sec   728 MBytes  6.10 Gbits/sec    0   2.73 MBytes
[  5]  28.00-29.00  sec   709 MBytes  5.95 Gbits/sec    0   2.73 MBytes
[  5]  29.00-30.00  sec   710 MBytes  5.96 Gbits/sec    0   2.73 MBytes
[  5]  30.00-31.00  sec   745 MBytes  6.25 Gbits/sec    0   2.73 MBytes
[  5]  31.00-32.00  sec   755 MBytes  6.33 Gbits/sec    0   2.73 MBytes
[  5]  32.00-33.00  sec   756 MBytes  6.34 Gbits/sec    0   2.73 MBytes
[  5]  33.00-34.00  sec   764 MBytes  6.41 Gbits/sec    0   2.73 MBytes
[  5]  34.00-35.00  sec   766 MBytes  6.43 Gbits/sec    0   2.73 MBytes
[  5]  35.00-36.00  sec   765 MBytes  6.42 Gbits/sec    0   2.73 MBytes
[  5]  36.00-37.00  sec   745 MBytes  6.25 Gbits/sec    0   2.73 MBytes
[  5]  37.00-38.00  sec   771 MBytes  6.47 Gbits/sec    0   2.73 MBytes
[  5]  38.00-39.00  sec   778 MBytes  6.52 Gbits/sec    0   2.73 MBytes
[  5]  39.00-40.00  sec   779 MBytes  6.53 Gbits/sec    0   2.73 MBytes
[  5]  40.00-41.00  sec   778 MBytes  6.52 Gbits/sec    0   2.73 MBytes
[  5]  41.00-42.00  sec   784 MBytes  6.57 Gbits/sec    0   2.73 MBytes
[  5]  42.00-43.00  sec   790 MBytes  6.63 Gbits/sec    0   2.73 MBytes
[  5]  43.00-44.00  sec   788 MBytes  6.61 Gbits/sec    0   2.73 MBytes
[  5]  44.00-45.00  sec   814 MBytes  6.83 Gbits/sec    0   2.73 MBytes
[  5]  45.00-46.00  sec   818 MBytes  6.86 Gbits/sec    0   2.73 MBytes
[  5]  46.00-47.00  sec   772 MBytes  6.48 Gbits/sec    0   2.73 MBytes
[  5]  47.00-48.00  sec   774 MBytes  6.49 Gbits/sec    0   2.73 MBytes
[  5]  48.00-49.00  sec   770 MBytes  6.46 Gbits/sec    0   2.73 MBytes
[  5]  49.00-50.00  sec   771 MBytes  6.47 Gbits/sec    0   2.73 MBytes
[  5]  50.00-51.00  sec   770 MBytes  6.46 Gbits/sec    0   2.73 MBytes
[  5]  51.00-52.00  sec   770 MBytes  6.46 Gbits/sec    0   2.73 MBytes
[  5]  52.00-53.00  sec   769 MBytes  6.45 Gbits/sec    0   2.73 MBytes
[  5]  53.00-54.00  sec   768 MBytes  6.44 Gbits/sec    0   2.73 MBytes
[  5]  54.00-55.00  sec   772 MBytes  6.48 Gbits/sec    0   2.73 MBytes
[  5]  55.00-56.00  sec   779 MBytes  6.53 Gbits/sec    0   2.73 MBytes
[  5]  56.00-57.00  sec   749 MBytes  6.28 Gbits/sec    0   2.73 MBytes
[  5]  57.00-58.00  sec   769 MBytes  6.45 Gbits/sec    0   2.73 MBytes
[  5]  58.00-59.00  sec   765 MBytes  6.42 Gbits/sec    0   2.73 MBytes
[  5]  59.00-60.00  sec   769 MBytes  6.45 Gbits/sec    0   2.73 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec  44.4 GBytes  6.35 Gbits/sec    0             sender
[  5]   0.00-60.00  sec  44.3 GBytes  6.35 Gbits/sec                  receiver

iperf Done.
```
</details>



<details> <summary> without vhost-net + Host->VM </summary>

Server
```bash
 $ iperf3 --server --interval 1
warning: this system does not seem to support IPv6 - trying IPv4
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
Accepted connection from 192.168.92.1, port 58108
[  5] local 192.168.92.100 port 5201 connected to 192.168.92.1 port 58124
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  1.89 GBytes  16.2 Gbits/sec
[  5]   1.00-2.00   sec  1.89 GBytes  16.2 Gbits/sec
[  5]   2.00-3.00   sec  1.89 GBytes  16.2 Gbits/sec
[  5]   3.00-4.00   sec  1.89 GBytes  16.2 Gbits/sec
[  5]   4.00-5.00   sec  1.89 GBytes  16.3 Gbits/sec
[  5]   5.00-6.00   sec  1.89 GBytes  16.2 Gbits/sec
[  5]   6.00-7.00   sec  1.88 GBytes  16.2 Gbits/sec
[  5]   7.00-8.00   sec  1.88 GBytes  16.2 Gbits/sec
[  5]   8.00-9.00   sec  1.88 GBytes  16.2 Gbits/sec
[  5]   9.00-10.00  sec  1.88 GBytes  16.1 Gbits/sec
[  5]  10.00-11.00  sec  1.88 GBytes  16.2 Gbits/sec
[  5]  11.00-12.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  12.00-13.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  13.00-14.00  sec  1.89 GBytes  16.3 Gbits/sec
[  5]  14.00-15.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  15.00-16.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  16.00-17.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  17.00-18.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  18.00-19.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  19.00-20.00  sec  1.88 GBytes  16.2 Gbits/sec
[  5]  20.00-21.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  21.00-22.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  22.00-23.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  23.00-24.00  sec  1.89 GBytes  16.3 Gbits/sec
[  5]  24.00-25.00  sec  1.89 GBytes  16.3 Gbits/sec
[  5]  25.00-26.00  sec  1.88 GBytes  16.2 Gbits/sec
[  5]  26.00-27.00  sec  1.88 GBytes  16.2 Gbits/sec
[  5]  27.00-28.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  28.00-29.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  29.00-30.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  30.00-31.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  31.00-32.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  32.00-33.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  33.00-34.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  34.00-35.00  sec  1.89 GBytes  16.3 Gbits/sec
[  5]  35.00-36.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  36.00-37.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  37.00-38.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  38.00-39.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  39.00-40.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  40.00-41.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  41.00-42.00  sec  1.88 GBytes  16.2 Gbits/sec
[  5]  42.00-43.00  sec  1.88 GBytes  16.2 Gbits/sec
[  5]  43.00-44.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  44.00-45.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  45.00-46.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  46.00-47.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  47.00-48.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  48.00-49.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  49.00-50.00  sec  1.88 GBytes  16.1 Gbits/sec
[  5]  50.00-51.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  51.00-52.00  sec  1.89 GBytes  16.3 Gbits/sec
[  5]  52.00-53.00  sec  1.89 GBytes  16.3 Gbits/sec
[  5]  53.00-54.00  sec  1.89 GBytes  16.3 Gbits/sec
[  5]  54.00-55.00  sec  1.88 GBytes  16.2 Gbits/sec
[  5]  55.00-56.00  sec  1.88 GBytes  16.2 Gbits/sec
[  5]  56.00-57.00  sec  1.88 GBytes  16.2 Gbits/sec
[  5]  57.00-58.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  58.00-59.00  sec  1.88 GBytes  16.2 Gbits/sec
[  5]  59.00-60.00  sec  1.89 GBytes  16.2 Gbits/sec
[  5]  60.00-60.00  sec  2.62 MBytes  15.4 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-60.00  sec   113 GBytes  16.2 Gbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201 (test #2)
-----------------------------------------------------------
^Ciperf3: interrupt - the server has terminated
```
Client
```bash
$ iperf3 --client 192.168.92.100 --interval 1 --time 60
Connecting to host 192.168.92.100, port 5201
[  5] local 192.168.92.1 port 58124 connected to 192.168.92.100 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]   1.00-2.00   sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]   2.00-3.00   sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]   3.00-4.00   sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]   4.00-5.00   sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]   5.00-6.00   sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]   6.00-7.00   sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]   7.00-8.00   sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]   8.00-9.00   sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]   9.00-10.00  sec  1.88 GBytes  16.1 Gbits/sec    0   3.06 MBytes
[  5]  10.00-11.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  11.00-12.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  12.00-13.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  13.00-14.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  14.00-15.00  sec  1.90 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  15.00-16.00  sec  1.90 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  16.00-17.00  sec  1.90 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  17.00-18.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  18.00-19.00  sec  1.90 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  19.00-20.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  20.00-21.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  21.00-22.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  22.00-23.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  23.00-24.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  24.00-25.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  25.00-26.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  26.00-27.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  27.00-28.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  28.00-29.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  29.00-30.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  30.00-31.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  31.00-32.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  32.00-33.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  33.00-34.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  34.00-35.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  35.00-36.00  sec  1.90 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  36.00-37.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  37.00-38.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  38.00-39.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  39.00-40.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  40.00-41.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  41.00-42.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  42.00-43.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  43.00-44.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  44.00-45.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  45.00-46.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  46.00-47.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  47.00-48.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  48.00-49.00  sec  1.90 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  49.00-50.00  sec  1.88 GBytes  16.1 Gbits/sec    0   3.06 MBytes
[  5]  50.00-51.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  51.00-52.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  52.00-53.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  53.00-54.00  sec  1.89 GBytes  16.3 Gbits/sec    0   3.06 MBytes
[  5]  54.00-55.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  55.00-56.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  56.00-57.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  57.00-58.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  58.00-59.00  sec  1.88 GBytes  16.2 Gbits/sec    0   3.06 MBytes
[  5]  59.00-60.00  sec  1.89 GBytes  16.2 Gbits/sec    0   3.06 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec   113 GBytes  16.2 Gbits/sec    0             sender
[  5]   0.00-60.00  sec   113 GBytes  16.2 Gbits/sec                  receiver

iperf Done.
```
</details>


<details> <summary> without vhost-net + VM->Host </summary>

Server
```bash
$ iperf3 --server --interval 1
warning: this system does not seem to support IPv6 - trying IPv4
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
Accepted connection from 192.168.92.100, port 40206
[  5] local 192.168.92.1 port 5201 connected to 192.168.92.100 port 40212
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  2.02 GBytes  17.3 Gbits/sec
[  5]   1.00-2.00   sec  2.03 GBytes  17.5 Gbits/sec
[  5]   2.00-3.00   sec  1.99 GBytes  17.1 Gbits/sec
[  5]   3.00-4.00   sec  2.02 GBytes  17.4 Gbits/sec
[  5]   4.00-5.00   sec  2.02 GBytes  17.4 Gbits/sec
[  5]   5.00-6.00   sec  2.02 GBytes  17.3 Gbits/sec
[  5]   6.00-7.00   sec  2.02 GBytes  17.3 Gbits/sec
[  5]   7.00-8.00   sec  2.02 GBytes  17.3 Gbits/sec
[  5]   8.00-9.00   sec  2.03 GBytes  17.4 Gbits/sec
[  5]   9.00-10.00  sec  2.02 GBytes  17.4 Gbits/sec
[  5]  10.00-11.00  sec  2.02 GBytes  17.4 Gbits/sec
[  5]  11.00-12.00  sec  2.02 GBytes  17.4 Gbits/sec
[  5]  12.00-13.00  sec  2.03 GBytes  17.4 Gbits/sec
[  5]  13.00-14.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  14.00-15.00  sec  2.09 GBytes  18.0 Gbits/sec
[  5]  15.00-16.00  sec  2.09 GBytes  17.9 Gbits/sec
[  5]  16.00-17.00  sec  2.08 GBytes  17.9 Gbits/sec
[  5]  17.00-18.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  18.00-19.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  19.00-20.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  20.00-21.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  21.00-22.00  sec  2.07 GBytes  17.8 Gbits/sec
[  5]  22.00-23.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  23.00-24.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  24.00-25.00  sec  2.07 GBytes  17.7 Gbits/sec
[  5]  25.00-26.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  26.00-27.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  27.00-28.00  sec  2.07 GBytes  17.8 Gbits/sec
[  5]  28.00-29.00  sec  2.08 GBytes  17.8 Gbits/sec
[  5]  29.00-30.00  sec  2.08 GBytes  17.8 Gbits/sec
[  5]  30.00-31.00  sec  2.07 GBytes  17.8 Gbits/sec
[  5]  31.00-32.00  sec  2.07 GBytes  17.8 Gbits/sec
[  5]  32.00-33.00  sec  2.08 GBytes  17.8 Gbits/sec
[  5]  33.00-34.00  sec  2.08 GBytes  17.9 Gbits/sec
[  5]  34.00-35.00  sec  2.07 GBytes  17.8 Gbits/sec
[  5]  35.00-36.00  sec  2.08 GBytes  17.8 Gbits/sec
[  5]  36.00-37.00  sec  2.08 GBytes  17.9 Gbits/sec
[  5]  37.00-38.00  sec  2.03 GBytes  17.5 Gbits/sec
[  5]  38.00-39.00  sec  2.03 GBytes  17.4 Gbits/sec
[  5]  39.00-40.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  40.00-41.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  41.00-42.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  42.00-43.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  43.00-44.00  sec  2.08 GBytes  17.8 Gbits/sec
[  5]  44.00-45.00  sec  2.07 GBytes  17.8 Gbits/sec
[  5]  45.00-46.00  sec  2.07 GBytes  17.8 Gbits/sec
[  5]  46.00-47.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  47.00-48.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  48.00-49.00  sec  2.01 GBytes  17.3 Gbits/sec
[  5]  49.00-50.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  50.00-51.00  sec  2.09 GBytes  18.0 Gbits/sec
[  5]  51.00-52.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  52.00-53.00  sec  2.07 GBytes  17.7 Gbits/sec
[  5]  53.00-54.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  54.00-55.00  sec  2.07 GBytes  17.8 Gbits/sec
[  5]  55.00-56.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  56.00-57.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  57.00-58.00  sec  2.06 GBytes  17.7 Gbits/sec
[  5]  58.00-59.00  sec  2.08 GBytes  17.8 Gbits/sec
[  5]  59.00-60.00  sec  2.07 GBytes  17.8 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-60.00  sec   123 GBytes  17.7 Gbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201 (test #2)
-----------------------------------------------------------
^Ciperf3: interrupt - the server has terminated
```
Client
```bash
$ iperf3 --client 192.168.92.1 --interval 1 --time 60
Connecting to host 192.168.92.1, port 5201
[  5] local 192.168.92.100 port 40212 connected to 192.168.92.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  2.02 GBytes  17.3 Gbits/sec    0   3.14 MBytes
[  5]   1.00-2.00   sec  2.03 GBytes  17.5 Gbits/sec    0   3.14 MBytes
[  5]   2.00-3.00   sec  1.99 GBytes  17.1 Gbits/sec    0   3.14 MBytes
[  5]   3.00-4.00   sec  2.02 GBytes  17.4 Gbits/sec    0   3.14 MBytes
[  5]   4.00-5.00   sec  2.02 GBytes  17.4 Gbits/sec    0   3.14 MBytes
[  5]   5.00-6.00   sec  2.02 GBytes  17.3 Gbits/sec    0   3.14 MBytes
[  5]   6.00-7.00   sec  2.02 GBytes  17.3 Gbits/sec    0   3.14 MBytes
[  5]   7.00-8.00   sec  2.02 GBytes  17.3 Gbits/sec    0   3.14 MBytes
[  5]   8.00-9.00   sec  2.03 GBytes  17.4 Gbits/sec    0   3.14 MBytes
[  5]   9.00-10.00  sec  2.02 GBytes  17.4 Gbits/sec    0   3.14 MBytes
[  5]  10.00-11.00  sec  2.02 GBytes  17.4 Gbits/sec    0   3.14 MBytes
[  5]  11.00-12.00  sec  2.02 GBytes  17.3 Gbits/sec    0   3.14 MBytes
[  5]  12.00-13.00  sec  2.03 GBytes  17.4 Gbits/sec    0   3.14 MBytes
[  5]  13.00-14.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.14 MBytes
[  5]  14.00-15.00  sec  2.09 GBytes  17.9 Gbits/sec    0   3.14 MBytes
[  5]  15.00-16.00  sec  2.09 GBytes  18.0 Gbits/sec    0   3.14 MBytes
[  5]  16.00-17.00  sec  2.08 GBytes  17.9 Gbits/sec    0   3.14 MBytes
[  5]  17.00-18.00  sec  2.07 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  18.00-19.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  19.00-20.00  sec  2.07 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  20.00-21.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  21.00-22.00  sec  2.07 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  22.00-23.00  sec  2.07 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  23.00-24.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  24.00-25.00  sec  2.07 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  25.00-26.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  26.00-27.00  sec  2.07 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  27.00-28.00  sec  2.07 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  28.00-29.00  sec  2.08 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  29.00-30.00  sec  2.08 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  30.00-31.00  sec  2.08 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  31.00-32.00  sec  2.07 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  32.00-33.00  sec  2.08 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  33.00-34.00  sec  2.08 GBytes  17.9 Gbits/sec    0   3.14 MBytes
[  5]  34.00-35.00  sec  2.08 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  35.00-36.00  sec  2.08 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  36.00-37.00  sec  2.08 GBytes  17.9 Gbits/sec    0   3.14 MBytes
[  5]  37.00-38.00  sec  2.03 GBytes  17.5 Gbits/sec    0   3.14 MBytes
[  5]  38.00-39.00  sec  2.03 GBytes  17.4 Gbits/sec    0   3.14 MBytes
[  5]  39.00-40.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.14 MBytes
[  5]  40.00-41.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  41.00-42.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  42.00-43.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  43.00-44.00  sec  2.07 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  44.00-45.00  sec  2.07 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  45.00-46.00  sec  2.07 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  46.00-47.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  47.00-48.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  48.00-49.00  sec  2.01 GBytes  17.3 Gbits/sec    0   3.14 MBytes
[  5]  49.00-50.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  50.00-51.00  sec  2.09 GBytes  18.0 Gbits/sec    0   3.14 MBytes
[  5]  51.00-52.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  52.00-53.00  sec  2.07 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  53.00-54.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  54.00-55.00  sec  2.07 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  55.00-56.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  56.00-57.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  57.00-58.00  sec  2.06 GBytes  17.7 Gbits/sec    0   3.14 MBytes
[  5]  58.00-59.00  sec  2.08 GBytes  17.8 Gbits/sec    0   3.14 MBytes
[  5]  59.00-60.00  sec  2.07 GBytes  17.8 Gbits/sec    0   3.14 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec   123 GBytes  17.7 Gbits/sec    0             sender
[  5]   0.00-60.00  sec   123 GBytes  17.7 Gbits/sec                  receiver

iperf Done.
```
</details>


<details> <summary> without vhost-net + VM0->VM1 </summary>

Server
```bash
$ iperf3 --server --interval 1
warning: this system does not seem to support IPv6 - trying IPv4
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
Accepted connection from 192.168.92.100, port 47346
[  5] local 192.168.92.101 port 5201 connected to 192.168.92.100 port 47348
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  1.47 GBytes  12.6 Gbits/sec
[  5]   1.00-2.00   sec  1.48 GBytes  12.7 Gbits/sec
[  5]   2.00-3.00   sec  1.47 GBytes  12.7 Gbits/sec
[  5]   3.00-4.00   sec  1.47 GBytes  12.7 Gbits/sec
[  5]   4.00-5.00   sec  1.47 GBytes  12.6 Gbits/sec
[  5]   5.00-6.00   sec  1.47 GBytes  12.6 Gbits/sec
[  5]   6.00-7.00   sec  1.47 GBytes  12.6 Gbits/sec
[  5]   7.00-8.00   sec  1.47 GBytes  12.7 Gbits/sec
[  5]   8.00-9.00   sec  1.48 GBytes  12.7 Gbits/sec
[  5]   9.00-10.00  sec  1.47 GBytes  12.6 Gbits/sec
[  5]  10.00-11.00  sec  1.47 GBytes  12.6 Gbits/sec
[  5]  11.00-12.00  sec  1.47 GBytes  12.6 Gbits/sec
[  5]  12.00-13.00  sec  1.47 GBytes  12.6 Gbits/sec
[  5]  13.00-14.00  sec  1.47 GBytes  12.6 Gbits/sec
[  5]  14.00-15.00  sec  1.47 GBytes  12.6 Gbits/sec
[  5]  15.00-16.00  sec  1.47 GBytes  12.6 Gbits/sec
[  5]  16.00-17.00  sec  1.47 GBytes  12.6 Gbits/sec
[  5]  17.00-18.00  sec  1.47 GBytes  12.6 Gbits/sec
[  5]  18.00-19.00  sec  1.48 GBytes  12.7 Gbits/sec
[  5]  19.00-20.00  sec  1.48 GBytes  12.7 Gbits/sec
[  5]  20.00-21.00  sec  1.47 GBytes  12.7 Gbits/sec
[  5]  21.00-22.00  sec  1.58 GBytes  13.6 Gbits/sec
[  5]  22.00-23.00  sec  1.49 GBytes  12.8 Gbits/sec
[  5]  23.00-24.00  sec  2.04 GBytes  17.5 Gbits/sec
[  5]  24.00-25.00  sec  2.08 GBytes  17.8 Gbits/sec
[  5]  25.00-26.00  sec  2.16 GBytes  18.5 Gbits/sec
[  5]  26.00-27.00  sec  2.32 GBytes  20.0 Gbits/sec
[  5]  27.00-28.00  sec  2.32 GBytes  19.9 Gbits/sec
[  5]  28.00-29.00  sec  2.33 GBytes  20.0 Gbits/sec
[  5]  29.00-30.00  sec  2.34 GBytes  20.1 Gbits/sec
[  5]  30.00-31.00  sec  2.34 GBytes  20.1 Gbits/sec
[  5]  31.00-32.00  sec  2.32 GBytes  19.9 Gbits/sec
[  5]  32.00-33.00  sec  1.80 GBytes  15.4 Gbits/sec
[  5]  33.00-34.00  sec  1.73 GBytes  14.9 Gbits/sec
[  5]  34.00-35.00  sec  1.97 GBytes  17.0 Gbits/sec
[  5]  35.00-36.00  sec  1.76 GBytes  15.1 Gbits/sec
[  5]  36.00-37.00  sec  1.73 GBytes  14.9 Gbits/sec
[  5]  37.00-38.00  sec  1.90 GBytes  16.3 Gbits/sec
[  5]  38.00-39.00  sec  1.78 GBytes  15.3 Gbits/sec
[  5]  39.00-40.00  sec  1.78 GBytes  15.3 Gbits/sec
[  5]  40.00-41.00  sec  1.80 GBytes  15.5 Gbits/sec
[  5]  41.00-42.00  sec  1.78 GBytes  15.3 Gbits/sec
[  5]  42.00-43.00  sec  1.80 GBytes  15.5 Gbits/sec
[  5]  43.00-44.00  sec  1.86 GBytes  16.0 Gbits/sec
[  5]  44.00-45.00  sec  1.87 GBytes  16.0 Gbits/sec
[  5]  45.00-46.00  sec  1.86 GBytes  16.0 Gbits/sec
[  5]  46.00-47.00  sec  1.86 GBytes  16.0 Gbits/sec
[  5]  47.00-48.00  sec  1.86 GBytes  16.0 Gbits/sec
[  5]  48.00-49.00  sec  1.87 GBytes  16.0 Gbits/sec
[  5]  49.00-50.00  sec  1.85 GBytes  15.9 Gbits/sec
[  5]  50.00-51.00  sec  2.25 GBytes  19.4 Gbits/sec
[  5]  51.00-52.00  sec  2.32 GBytes  19.9 Gbits/sec
[  5]  52.00-53.00  sec  2.16 GBytes  18.6 Gbits/sec
[  5]  53.00-54.00  sec  1.82 GBytes  15.6 Gbits/sec
[  5]  54.00-55.00  sec  1.81 GBytes  15.5 Gbits/sec
[  5]  55.00-56.00  sec  1.81 GBytes  15.5 Gbits/sec
[  5]  56.00-57.00  sec  1.80 GBytes  15.5 Gbits/sec
[  5]  57.00-58.00  sec  1.73 GBytes  14.9 Gbits/sec
[  5]  58.00-59.00  sec  1.73 GBytes  14.8 Gbits/sec
[  5]  59.00-60.00  sec  2.08 GBytes  17.9 Gbits/sec
[  5]  60.00-60.00  sec  1.25 MBytes  16.4 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-60.00  sec   107 GBytes  15.3 Gbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201 (test #2)
-----------------------------------------------------------
^Ciperf3: interrupt - the server has terminated
```
Client
```bash
 $ iperf3 --client 192.168.92.101 --interval 1 --time 60
Connecting to host 192.168.92.101, port 5201
[  5] local 192.168.92.100 port 47348 connected to 192.168.92.101 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]   1.00-2.00   sec  1.48 GBytes  12.7 Gbits/sec    0   3.09 MBytes
[  5]   2.00-3.00   sec  1.48 GBytes  12.7 Gbits/sec    0   3.09 MBytes
[  5]   3.00-4.00   sec  1.47 GBytes  12.7 Gbits/sec    0   3.09 MBytes
[  5]   4.00-5.00   sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]   5.00-6.00   sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]   6.00-7.00   sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]   7.00-8.00   sec  1.47 GBytes  12.7 Gbits/sec    0   3.09 MBytes
[  5]   8.00-9.00   sec  1.47 GBytes  12.7 Gbits/sec    0   3.09 MBytes
[  5]   9.00-10.00  sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]  10.00-11.00  sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]  11.00-12.00  sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]  12.00-13.00  sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]  13.00-14.00  sec  1.47 GBytes  12.7 Gbits/sec    0   3.09 MBytes
[  5]  14.00-15.00  sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]  15.00-16.00  sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]  16.00-17.00  sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]  17.00-18.00  sec  1.47 GBytes  12.6 Gbits/sec    0   3.09 MBytes
[  5]  18.00-19.00  sec  1.48 GBytes  12.7 Gbits/sec    0   3.09 MBytes
[  5]  19.00-20.00  sec  1.48 GBytes  12.7 Gbits/sec    0   3.09 MBytes
[  5]  20.00-21.00  sec  1.47 GBytes  12.7 Gbits/sec    0   3.09 MBytes
[  5]  21.00-22.00  sec  1.58 GBytes  13.6 Gbits/sec    0   3.09 MBytes
[  5]  22.00-23.00  sec  1.49 GBytes  12.8 Gbits/sec    0   3.09 MBytes
[  5]  23.00-24.00  sec  2.04 GBytes  17.5 Gbits/sec    0   3.09 MBytes
[  5]  24.00-25.00  sec  2.08 GBytes  17.8 Gbits/sec    0   3.09 MBytes
[  5]  25.00-26.00  sec  2.16 GBytes  18.5 Gbits/sec    0   3.09 MBytes
[  5]  26.00-27.00  sec  2.32 GBytes  20.0 Gbits/sec    0   3.09 MBytes
[  5]  27.00-28.00  sec  2.32 GBytes  19.9 Gbits/sec    0   3.09 MBytes
[  5]  28.00-29.00  sec  2.33 GBytes  20.0 Gbits/sec    0   3.09 MBytes
[  5]  29.00-30.00  sec  2.34 GBytes  20.1 Gbits/sec    0   3.09 MBytes
[  5]  30.00-31.00  sec  2.34 GBytes  20.1 Gbits/sec    0   3.09 MBytes
[  5]  31.00-32.00  sec  2.32 GBytes  19.9 Gbits/sec    0   3.09 MBytes
[  5]  32.00-33.00  sec  1.80 GBytes  15.4 Gbits/sec    0   3.09 MBytes
[  5]  33.00-34.00  sec  1.73 GBytes  14.9 Gbits/sec    0   3.09 MBytes
[  5]  34.00-35.00  sec  1.97 GBytes  17.0 Gbits/sec    0   3.09 MBytes
[  5]  35.00-36.00  sec  1.76 GBytes  15.1 Gbits/sec    0   3.09 MBytes
[  5]  36.00-37.00  sec  1.73 GBytes  14.9 Gbits/sec    0   3.09 MBytes
[  5]  37.00-38.00  sec  1.90 GBytes  16.3 Gbits/sec    0   3.09 MBytes
[  5]  38.00-39.00  sec  1.78 GBytes  15.3 Gbits/sec    0   3.09 MBytes
[  5]  39.00-40.00  sec  1.78 GBytes  15.3 Gbits/sec    0   3.09 MBytes
[  5]  40.00-41.00  sec  1.80 GBytes  15.5 Gbits/sec    0   3.09 MBytes
[  5]  41.00-42.00  sec  1.78 GBytes  15.3 Gbits/sec    0   3.09 MBytes
[  5]  42.00-43.00  sec  1.80 GBytes  15.5 Gbits/sec    0   3.09 MBytes
[  5]  43.00-44.00  sec  1.86 GBytes  16.0 Gbits/sec    0   3.09 MBytes
[  5]  44.00-45.00  sec  1.87 GBytes  16.0 Gbits/sec    0   3.09 MBytes
[  5]  45.00-46.00  sec  1.86 GBytes  16.0 Gbits/sec    0   3.09 MBytes
[  5]  46.00-47.00  sec  1.86 GBytes  16.0 Gbits/sec    0   3.09 MBytes
[  5]  47.00-48.00  sec  1.86 GBytes  16.0 Gbits/sec    0   3.09 MBytes
[  5]  48.00-49.00  sec  1.87 GBytes  16.0 Gbits/sec    0   3.09 MBytes
[  5]  49.00-50.00  sec  1.85 GBytes  15.9 Gbits/sec    0   3.09 MBytes
[  5]  50.00-51.00  sec  2.25 GBytes  19.4 Gbits/sec    0   3.09 MBytes
[  5]  51.00-52.00  sec  2.32 GBytes  19.9 Gbits/sec    0   3.09 MBytes
[  5]  52.00-53.00  sec  2.16 GBytes  18.6 Gbits/sec    0   3.09 MBytes
[  5]  53.00-54.00  sec  1.82 GBytes  15.6 Gbits/sec    0   3.09 MBytes
[  5]  54.00-55.00  sec  1.81 GBytes  15.5 Gbits/sec    0   3.09 MBytes
[  5]  55.00-56.00  sec  1.81 GBytes  15.5 Gbits/sec    0   3.09 MBytes
[  5]  56.00-57.00  sec  1.81 GBytes  15.5 Gbits/sec    0   3.09 MBytes
[  5]  57.00-58.00  sec  1.73 GBytes  14.9 Gbits/sec    0   3.09 MBytes
[  5]  58.00-59.00  sec  1.73 GBytes  14.9 Gbits/sec    0   3.09 MBytes
[  5]  59.00-60.00  sec  2.08 GBytes  17.9 Gbits/sec    0   3.09 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec   107 GBytes  15.3 Gbits/sec    0             sender
[  5]   0.00-60.00  sec   107 GBytes  15.3 Gbits/sec                  receiver

iperf Done.
```
</details>


<details> <summary> without vhost-net + VM1->VM0 </summary>

Server
```bash
$ iperf3 --server --interval 1
warning: this system does not seem to support IPv6 - trying IPv4
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
Accepted connection from 192.168.92.101, port 42612
[  5] local 192.168.92.100 port 5201 connected to 192.168.92.101 port 42614
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  1.45 GBytes  12.4 Gbits/sec
[  5]   1.00-2.00   sec  1.43 GBytes  12.3 Gbits/sec
[  5]   2.00-3.00   sec  1.44 GBytes  12.4 Gbits/sec
[  5]   3.00-4.00   sec  1.44 GBytes  12.4 Gbits/sec
[  5]   4.00-5.00   sec  1.45 GBytes  12.4 Gbits/sec
[  5]   5.00-6.00   sec  1.45 GBytes  12.4 Gbits/sec
[  5]   6.00-7.00   sec  1.46 GBytes  12.6 Gbits/sec
[  5]   7.00-8.00   sec  1.47 GBytes  12.7 Gbits/sec
[  5]   8.00-9.00   sec  1.48 GBytes  12.7 Gbits/sec
[  5]   9.00-10.00  sec  1.49 GBytes  12.8 Gbits/sec
[  5]  10.00-11.00  sec  2.05 GBytes  17.6 Gbits/sec
[  5]  11.00-12.00  sec  2.31 GBytes  19.8 Gbits/sec
[  5]  12.00-13.00  sec  2.29 GBytes  19.6 Gbits/sec
[  5]  13.00-14.00  sec  2.33 GBytes  20.0 Gbits/sec
[  5]  14.00-15.00  sec  2.28 GBytes  19.6 Gbits/sec
[  5]  15.00-16.00  sec  2.22 GBytes  19.0 Gbits/sec
[  5]  16.00-17.00  sec  2.29 GBytes  19.7 Gbits/sec
[  5]  17.00-18.00  sec  2.31 GBytes  19.9 Gbits/sec
[  5]  18.00-19.00  sec  2.34 GBytes  20.1 Gbits/sec
[  5]  19.00-20.00  sec  2.35 GBytes  20.2 Gbits/sec
[  5]  20.00-21.00  sec  2.34 GBytes  20.1 Gbits/sec
[  5]  21.00-22.00  sec  2.35 GBytes  20.1 Gbits/sec
[  5]  22.00-23.00  sec  2.35 GBytes  20.2 Gbits/sec
[  5]  23.00-24.00  sec  2.35 GBytes  20.2 Gbits/sec
[  5]  24.00-25.00  sec  2.35 GBytes  20.2 Gbits/sec
[  5]  25.00-26.00  sec  2.35 GBytes  20.2 Gbits/sec
[  5]  26.00-27.00  sec  2.41 GBytes  20.7 Gbits/sec
[  5]  27.00-28.00  sec  2.41 GBytes  20.7 Gbits/sec
[  5]  28.00-29.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  29.00-30.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  30.00-31.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  31.00-32.00  sec  1.73 GBytes  14.9 Gbits/sec
[  5]  32.00-33.00  sec  1.79 GBytes  15.3 Gbits/sec
[  5]  33.00-34.00  sec  1.78 GBytes  15.3 Gbits/sec
[  5]  34.00-35.00  sec  1.77 GBytes  15.2 Gbits/sec
[  5]  35.00-36.00  sec  1.71 GBytes  14.7 Gbits/sec
[  5]  36.00-37.00  sec  1.70 GBytes  14.6 Gbits/sec
[  5]  37.00-38.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  38.00-39.00  sec  1.73 GBytes  14.9 Gbits/sec
[  5]  39.00-40.00  sec  1.73 GBytes  14.8 Gbits/sec
[  5]  40.00-41.00  sec  1.73 GBytes  14.9 Gbits/sec
[  5]  41.00-42.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  42.00-43.00  sec  1.71 GBytes  14.7 Gbits/sec
[  5]  43.00-44.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  44.00-45.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  45.00-46.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  46.00-47.00  sec  1.71 GBytes  14.7 Gbits/sec
[  5]  47.00-48.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  48.00-49.00  sec  1.73 GBytes  14.8 Gbits/sec
[  5]  49.00-50.00  sec  1.73 GBytes  14.8 Gbits/sec
[  5]  50.00-51.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  51.00-52.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  52.00-53.00  sec  1.73 GBytes  14.8 Gbits/sec
[  5]  53.00-54.00  sec  1.73 GBytes  14.8 Gbits/sec
[  5]  54.00-55.00  sec  1.73 GBytes  14.8 Gbits/sec
[  5]  55.00-56.00  sec  1.71 GBytes  14.7 Gbits/sec
[  5]  56.00-57.00  sec  1.71 GBytes  14.7 Gbits/sec
[  5]  57.00-58.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  58.00-59.00  sec  1.73 GBytes  14.8 Gbits/sec
[  5]  59.00-60.00  sec  1.72 GBytes  14.8 Gbits/sec
[  5]  60.00-60.00  sec  1.50 MBytes  12.2 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-60.00  sec   112 GBytes  16.0 Gbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201 (test #2)
-----------------------------------------------------------
^Ciperf3: interrupt - the server has terminated
```
Client
```bash
$ iperf3 --client 192.168.92.100 --interval 1 --time 60
Connecting to host 192.168.92.100, port 5201
[  5] local 192.168.92.101 port 42614 connected to 192.168.92.100 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  1.45 GBytes  12.4 Gbits/sec    0   3.11 MBytes
[  5]   1.00-2.00   sec  1.43 GBytes  12.3 Gbits/sec    0   3.11 MBytes
[  5]   2.00-3.00   sec  1.44 GBytes  12.4 Gbits/sec    0   3.11 MBytes
[  5]   3.00-4.00   sec  1.45 GBytes  12.4 Gbits/sec    0   3.11 MBytes
[  5]   4.00-5.00   sec  1.45 GBytes  12.4 Gbits/sec    0   3.11 MBytes
[  5]   5.00-6.00   sec  1.45 GBytes  12.4 Gbits/sec    0   3.11 MBytes
[  5]   6.00-7.00   sec  1.46 GBytes  12.6 Gbits/sec    0   3.11 MBytes
[  5]   7.00-8.00   sec  1.47 GBytes  12.6 Gbits/sec    0   3.11 MBytes
[  5]   8.00-9.00   sec  1.48 GBytes  12.7 Gbits/sec    0   3.11 MBytes
[  5]   9.00-10.00  sec  1.49 GBytes  12.8 Gbits/sec    0   3.11 MBytes
[  5]  10.00-11.00  sec  2.05 GBytes  17.6 Gbits/sec    0   3.11 MBytes
[  5]  11.00-12.00  sec  2.31 GBytes  19.8 Gbits/sec    0   3.11 MBytes
[  5]  12.00-13.00  sec  2.29 GBytes  19.6 Gbits/sec    0   3.11 MBytes
[  5]  13.00-14.00  sec  2.33 GBytes  20.0 Gbits/sec    0   3.11 MBytes
[  5]  14.00-15.00  sec  2.28 GBytes  19.6 Gbits/sec    0   3.11 MBytes
[  5]  15.00-16.00  sec  2.22 GBytes  19.0 Gbits/sec    0   3.11 MBytes
[  5]  16.00-17.00  sec  2.29 GBytes  19.7 Gbits/sec    0   3.11 MBytes
[  5]  17.00-18.00  sec  2.32 GBytes  19.9 Gbits/sec    0   3.11 MBytes
[  5]  18.00-19.00  sec  2.34 GBytes  20.1 Gbits/sec    0   3.11 MBytes
[  5]  19.00-20.00  sec  2.35 GBytes  20.2 Gbits/sec    0   3.11 MBytes
[  5]  20.00-21.00  sec  2.34 GBytes  20.1 Gbits/sec    0   3.11 MBytes
[  5]  21.00-22.00  sec  2.34 GBytes  20.1 Gbits/sec    0   3.11 MBytes
[  5]  22.00-23.00  sec  2.35 GBytes  20.2 Gbits/sec    0   3.11 MBytes
[  5]  23.00-24.00  sec  2.35 GBytes  20.2 Gbits/sec    0   3.11 MBytes
[  5]  24.00-25.00  sec  2.35 GBytes  20.2 Gbits/sec    0   3.11 MBytes
[  5]  25.00-26.00  sec  2.35 GBytes  20.2 Gbits/sec    0   3.11 MBytes
[  5]  26.00-27.00  sec  2.41 GBytes  20.7 Gbits/sec    0   3.11 MBytes
[  5]  27.00-28.00  sec  2.41 GBytes  20.7 Gbits/sec    0   3.11 MBytes
[  5]  28.00-29.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  29.00-30.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  30.00-31.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  31.00-32.00  sec  1.73 GBytes  14.9 Gbits/sec    0   3.11 MBytes
[  5]  32.00-33.00  sec  1.79 GBytes  15.3 Gbits/sec    0   3.11 MBytes
[  5]  33.00-34.00  sec  1.78 GBytes  15.3 Gbits/sec    0   3.11 MBytes
[  5]  34.00-35.00  sec  1.77 GBytes  15.2 Gbits/sec    0   3.11 MBytes
[  5]  35.00-36.00  sec  1.71 GBytes  14.7 Gbits/sec    0   3.11 MBytes
[  5]  36.00-37.00  sec  1.70 GBytes  14.6 Gbits/sec    0   3.11 MBytes
[  5]  37.00-38.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  38.00-39.00  sec  1.73 GBytes  14.9 Gbits/sec    0   3.11 MBytes
[  5]  39.00-40.00  sec  1.73 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  40.00-41.00  sec  1.73 GBytes  14.9 Gbits/sec    0   3.11 MBytes
[  5]  41.00-42.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  42.00-43.00  sec  1.71 GBytes  14.7 Gbits/sec    0   3.11 MBytes
[  5]  43.00-44.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  44.00-45.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  45.00-46.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  46.00-47.00  sec  1.71 GBytes  14.7 Gbits/sec    0   3.11 MBytes
[  5]  47.00-48.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  48.00-49.00  sec  1.73 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  49.00-50.00  sec  1.73 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  50.00-51.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  51.00-52.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  52.00-53.00  sec  1.73 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  53.00-54.00  sec  1.73 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  54.00-55.00  sec  1.73 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  55.00-56.00  sec  1.71 GBytes  14.7 Gbits/sec    0   3.11 MBytes
[  5]  56.00-57.00  sec  1.71 GBytes  14.7 Gbits/sec    0   3.11 MBytes
[  5]  57.00-58.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  58.00-59.00  sec  1.73 GBytes  14.8 Gbits/sec    0   3.11 MBytes
[  5]  59.00-60.00  sec  1.72 GBytes  14.8 Gbits/sec    0   3.11 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-60.00  sec   112 GBytes  16.0 Gbits/sec    0             sender
[  5]   0.00-60.00  sec   112 GBytes  16.0 Gbits/sec                  receiver

iperf Done.
```
</details>

</details>



### Current balloon speed

```bash
[   19.069262] balloon: hetero size change has taken 10818601803
[   21.589000] balloon: size change has taken 13338485865
```

| Inflate | Time                             | Speed   |
| ------- | -------------------------------- | ------- |
| 4G      | 10818601803/2.4GHz=4.5077507512s | 1.59G/s |
| 6G      | 13338485865/2.4GHz=5.5577024438s | 1.08G/s |

 

## How can we demonstrate the overall performance?



## How can we pass memory access information to the host?

*Extending balloon statistic reporting.*

