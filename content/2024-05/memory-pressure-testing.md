+++
title = "Memory Pressure Testing"
date = "2024-05-16"
+++

## Define memory pressure?

### PSI in academia

The first thing on my mind when talking about pressure is the Pressure Stall Info (PSI) proposed by [ASPLOS22]TMO.

> PSI measure the lost work due the lack of a resource.
> The measurement is done by measure the time a process is in runnable or stalled state.

> What would trigger the stalled state?
> The paper did not give a detailed discussion.

From my own understanding, the pressure is closely related to memory allocation.
Pressure describes the free memory level.

#### Watermarks

There are also some predefined levels called watermarks.
`promo`, `high`, `low`, `min` watermarks exist natively in mordern linux as discussed in [ASPLOS23]TPP.
When free memory dips below the low watermark,
memory reclaim will start and will not stop until enough of memory is freed to reach the high watermark.
In the meantime, processes that need to allocate memory are stalled.
When free memory dips below min,
only the most essential kernel functionalities are allowed to allocate memory.
`promo` is the highest watermark.
TPP's tiered memory management tries to keep the free memory of the top tier around this watermark,
so that the top tier is nearly fully utilized while not introducing additional memory reclaimation overhead.

Calculation: (source: `__setup_per_zone_wmarks()`)
$$
M = \text{zone managed memory size} \\
M_l = \text{lowmem zone managed memory size} \\
m_0 = 1204 : \text{min watermark} \\
m_1 : \text{low watermark} \\
m_2 : \text{high watermark} \\
m_3 : \text{promo watermark} \\
m_b : \text{boost watermark} \\
m_{b0} = 0 : \text{initial boost watermark} \\
S_m =\frac{10}{10000}: \text{watermark scale} \\
S_b =\frac{15000}{10000} : \text{boot scale} \\
s : \text{step between watermark} \\

s = max\{ m_0 \times \frac{1}{4} \frac{M}{M_1}, M \times S_m\} \\
\approx max\{\frac{m_0}{4}, S_m M \} \ \ \ \ (M \to M_l) \\
= S_m M \ \ \ \ (S_m M \gg m_0) \\
m_i = m_0 + i \times s \ \ \ \  (i>0) \\
m_b = min\{m_{b0} + 512, max\{S_b m_2, 512\}\} \\
=\begin{cases}
m_{b0}    &S_b m_2 > 512 \\
512    &\text{else}
\end{cases} \\
= m_{b0} \ \ \ \ (S_b > 1, m_2 \gg 512) \\
= 0
$$


Let's verify this:

```python
# sudo drgn
>>> try:
...     prog.load_debug_info(["/tmp/vmlinux"])
... except MissingDebugInfoError as e:
...     print(f"Failed to load debug info: {e}")
...
>>> node_data = prog['node_data']
>>> node_states = prog['node_states']
>>> N_ONLINE = prog['N_ONLINE']
>>> node_states[N_ONLINE]
(nodemask_t){
        .bits = (unsigned long [16]){ 1 },
}
>>> len(node_data[0].node_zones)
4
>>> [z.name.string_() for z in node_data[0].node_zones]
[b'DMA', b'Normal', b'Movable', b'Device']
>>> [z.managed_pages.counter.value_() for z in node_data[0].node_zones]
[3840, 4068988, 0, 0]
>>> # M=4068988 M_l=3840+4068988 M->M_l
>>> node_data[0].node_zones[1]._watermark
(unsigned long [4]){ 4025, 8092, 12159, 16226 }
>>> 16226 - 12159 == 12159 - 8092 == 8092 - 4025
True
>>> 8092 - 4025
4067
>>> node_data[0].node_zones[1].managed_pages.counter.value_() * (10/1000)
4068.988
>>> # s ≈ S_m * M = 4068
>>> [z.watermark_boost.value_() for z in node_data[0].node_zones]
[0, 0, 0, 0]
>>> # Increase S_m by 10:
>>> # sudo sysctl -w vm.watermark_scale_factor=200
>>> node_data[0].node_zones[1]._watermark
(unsigned long [4]){ 4025, 44714, 85403, 126092 }
>>> 248162 - 166783 == 166783 - 85404 == 85404 - 4025
True
>>> 85404 - 4025
81379
>>> node_data[0].node_zones[1].managed_pages.counter.value_() * (200/10000)
81379.76
```

**Problems**

- Is TPP's `promo` watermark still valid under a virtualized environemt where balloon exists?
- Does PSI account for the impact of migration, such as process stalled when accessing a page under migration?
- Does the "lost work" include "stolen" CPU time by the memory reclaimation code?

#### Watermark Pressure Testing

To test the effect of memory pressure on userspace allocation, we set up the following test. First preallocate some pages to reach the desired memory pressure (watermark level), then repeatedly (100x) allocate and free blocks of memory so that the pressure is maintained (not reaching a higher or lower watermark level). [src](https://gist.github.com/vtta/5a848e2ed846aa66990d3166797d8a5a)

```
watermarks: min=4025 low=410923 high=817821 promo=1224719
bench baseline: starting...
prealloc: allocated=2174929 freeram=1625598 wmark=1631617
bench baseline: finished after 54s iteration took 546ms on average
bench promo: starting...
prealloc: allocated=2513400 freeram=1341705 wmark=1224719
...
prealloc: allocated=2631140 freeram=1224777 wmark=1224719
prealloc: allocated=2631198 freeram=1224714 wmark=1224719
bench promo: finished after 54s iteration took 547ms on average
bench high: starting...
prealloc: allocated=2885308 freeram=933229 wmark=817821
...
prealloc: allocated=3001708 freeram=816301 wmark=817821
bench high: finished after 54s iteration took 548ms on average
bench low: starting...
prealloc: allocated=3218981 freeram=505699 wmark=410923
prealloc: allocated=3313757 freeram=431485 wmark=410923
...
prealloc: allocated=3410906 freeram=409931 wmark=410923
bench low: finished after 56s iteration took 564ms on average
```

| Case     | Free Memory   | Average Iteration |
| -------- | ------------- | ----------------- |
| Baseline | [promo, ∞)    | 546ms             |
| Promo    | [high, promo) | 547ms             |
| High     | [low, high)   | 548ms             |
| Low      | [min, low)    | 564ms             |

Observation: 

- Only in the [min, low) case, the performance is affected. (~5%); 
- In the meantime, `kswapd0` kennel background thread spins up and utilises about 20~30% CPU and uses ~125MiB/s disk read and ~3MiB/s write read bandwidth.

Thoughts:
- When below `low` watermark, directly reclaim is activated to swap pages to disk. This is validated by the above results.
- When below `high`, asynchronous reclaim should be active, but I saw nothing in the `htop` monitor.
- When below `promo`, tiered memory management should be active, but the test environment only contains one node, we can change to a tiered configuration and redesign a set of experiments.

#### GUPS under S_m = 0.001/0.0125/0.05/0.1

- $S_m=0.1\%$: This is the default setting on vanilla Linux. The min/low/high/promo watermarks (normal zone) are around 10/30/45/60MiB.
  - DRAM: The DRAM free memory jumps around 45MiB. (DRAM normal zone high watermark) (although after accounting other zones, this might be the global low watermark) 
  - PMEM: For ours/TPP/AutoNUMA/Nothing,t he PMEM free memory (\~360-460MiB) is well above the promo watermark. Memtis is slightly lower but still above promo watermark.
  - `kswapd`: inactive in all designs
  - `pgmigrate`: Ours have the highest among all: 160K >> 9K (AutoNUMA, second highest)
  - `pgmajfault`: nearly the same across all designs: no swapping is  triggered
- $S_m=1.25\%$: This setting tries to increase pressure a little bit, by matching the actual free memory (\~400MiB) with low watermark (\~200MiB each normal zone). The resulting min/low/high/promo watermarks are around 10/220/420/620MiB.
  - Expectation: this should be the ideal case, pressure is just enough to trigger reclamation while not too large to trigger swapping IO.
  - DRAM
    - TPP/AutoNUMA/Nothing jumps between low and high watermark;
    - Ours is a little bit lower (why???);
    - Memtis jumps around 100MiB because of fixed watermark.
  - PMEM
    - Ours/TPP/AutoNUMA/Nothing is nearly a stright line around the low watermark.
    - Memtis jumps around 100MiB
  - `kswapd`
    - Only TPP/Nothing has `kswpad` active (only on DRAM node) for a significant period of time \~1300s out of 1300s+ benchmark runtime. 
    - TPP's `kswapd` shoule demote pages to PMEM; Nothing's `kswapd` should swap pages to disk. However, this is not what happened:
  - `pgmajfault`: Both TPP and Nothing have a bery high value of 2M+
  - `pgmigrate`: only ours has a significant value: 150K+
- $S_m=5\%$: This setting increase pressure further, making the sum of low watermark 10% of total memory (\~1.6G) matching the theoretical free memory (16G system RAM minus 14G workingset and other kernel usage)

### PSI in Linux kernel

#### Definition of [PSI](https://docs.kernel.org/accounting/psi.html)

> The psi feature identifies and quantifies the disruptions caused by resource crunches and the time impact it has on complex workloads or even entire systems.
> Pressure information for each resource is exported through the respective file in /proc/pressure/ -- cpu, memory, and io.
>
> ```
> some avg10=0.00 avg60=0.00 avg300=0.07 total=2888755
> full avg10=0.00 avg60=0.00 avg300=0.00 total=0
> ```
>
> The “some” line indicates the share of time in which at least some tasks are stalled on a given resource.
> The “full” line indicates the share of time in which all non-idle tasks are stalled on a given resource simultaneously.
> The ratios (in %) are tracked as recent trends over ten, sixty, and three hundred second windows, which gives insight into short term events as well as medium and long term trends.

Under high memory pressure, tasks that need to allocate memory or access pages under migration would be blocked. This would be accounted for by the PSI memory metric.

#### Test PSI metric

First enabling PSI reporting via kernel command-line arguments `psi=on`.
Then capturing the psi statistic every second via [`python3 pressure.py --resource memory`](https://gist.github.com/vtta/40b7033d5f41945ccab8b27bd421e77a) when running a benchmark (GUPS here) with a larger memory footprint than the DRAM node size.
We can see for Memtis/TPP/AutoNUMA/Vanilla kernel, nothing is stalled:

```csv
some,some,some,some,full,full,full,full
avg10,avg60,avg300,total,avg10,avg60,avg300,total
0.0,0.0,0.0,0,0.0,0.0,0.0,0
0.0,0.0,0.0,0,0.0,0.0,0.0,0
0.0,0.0,0.0,0,0.0,0.0,0.0,0
0.0,0.0,0.0,0,0.0,0.0,0.0,0
0.0,0.0,0.0,0,0.0,0.0,0.0,0
...
```

### The hidden Linux VM pressure
Under `mm/vmpressure.c` lies the old memory pressure code dating back to 2012.
Try checking if the code is working by bpftrace:
```bash
# sudo bpftrace -e ' kfunc:vmpressure { printf("vmpressure(gfp=0x%x,memcg=%p,tree=%d,scanned=%lu,reclaimed=%lu)\n", args->gfp, args->memcg, args->tree, args->scanned, args->reclaimed) } '
clear@clr ~ $ sudo bpftrace -e ' kfunc:vmpressure { printf("vmpressure(gfp=0x%x,memcg=%p,tree=%d,scanned=%lu,reclaimed=%lu)\n", args->gfp, args->memcg, args->tree, args->scanned, args->reclaimed) } '
Attaching 1 probe...

Broadcast message from root@clr (Wed 2024-05-22 07:14:24 UTC):

The system will power off now!
```
Nothing is captured during an entire benchmark run.
We can see the caller of `vmpressure()` is the memory reclaimation code,
include old `shrink_node()`, its subroutine `shrink_node_memcgs()` and the MGRLU's `lru_gen_shrink_node()`.
Try to trace these functions:
```bash
lear@clr ~ $ sudo bpftrace -e ' kfunc:shrink_node { printf(" shrink_node( pgdat=%p sc=%p )\n", args->pgdat, args->sc) } '
Attaching 1 probe...

Broadcast message from root@clr (Wed 2024-05-22 07:25:44 UTC):

The system will power off now!
```
Still nothing.
But this is understandable because only when there is not enough memory to satisfy allocation will reclamation be triggered.

