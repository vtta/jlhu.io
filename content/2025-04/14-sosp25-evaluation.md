+++

+++



- Breakdown study

  - Balloon: 

  - Access tracking scalability: motivation figure

  - Hotness classification accuracy: realtime GUPS

  - Page migration overhead: locking statistics (via try_to_unmap/try_to_migrate); kswapd cpu usage

    - ```bash
      ps -p $(pgrep kswapd) -o pid,pcpu,cputime,etimes,cmd --no-headers | awk '{print "PID: " $1 " | Average CPU: " ($3/$4*100) "% | Total CPU time: " $3 " seconds | Elapsed time: " $4 " seconds"}'
      
      ```

- Real-world study

- CXL study







### tmm 

### v1

```
=== Efficient and Scalable Guest-Delegated TMM
#include "figure/evaluation/breakdown.typ"
#include "figure/evaluation/realtime-gups.typ"

*EPT-friendly PEBS Access Tracking*
To provide insight into our guest-delegated TMM design's performance characteristics, we conducted a detailed overhead breakdown study presented in @fig-evaluation-breakdown.
We deployed nine concurrent virtual machines running the GUPS workload and measured the average CPU time consumed by various TMM stages across all VMs.

Demeter's innovative sample collection approach, which proactively drains samples during context switches, demonstrates great efficiency, consuming only approximately three seconds of total CPU time.
This represents a 16× reduction in overhead compared to Memtis, which relies on dedicated sample collection worker threads.
Other guest-based solutions (TPP and Nomad) depend on exhaustive page table walks and PTE.A/D bit scanning for all managed physical pages.
Even TPP, the most efficient among these alternatives, incurs 1.2× higher CPU overhead than Memtis while providing less accurate hotness information.
\
*Range-based Hotness Classification*
@fig-evaluation-realtime-gups illustrates the temporal performance characteristics of our range-based hotness classification through real-time GUPS throughput measurements under identical workload conditions.
As TMM systems identify hot memory regions and migrate them to FMEM, we expect to observe increasing instantaneous throughput, with peak values indicating classification accuracy and plateau timing reflecting detection speed.

Demeter demonstrates superior responsiveness with the steepest throughput increase during the initial 80 seconds, highlighting its ability to rapidly identify and prioritize hot data ranges.
The subsequent temporary throughput decline correlates with migration traffic as pages that are less hot are relocated between tiers.
Notably, Demeter achieves both the earliest completion time and highest maximum instantaneous throughput among all evaluated solutions, confirming both its classification accuracy and overall efficiency.
@fig-evaluation-breakdown further validates these results, showing that Demeter identifies the most comprehensive set of hot data while incurring only $2/3$ the overhead of the next best alternative.
\
*Balanced Page Relocation*
Demeter's migration mechanism demonstrates exceptional efficiency despite handling the largest volume of hot data (as evidenced in @fig-evaluation-realtime-gups).
As shown in @fig-evaluation-breakdown, our balanced page relocation implementation consumes only 28% of the overhead incurred by TPP, which spends nearly half its execution time on software-level page fault handler operations.
We deliberately exclude hardware-level page faulting overhead from this comparison due to measurement complexities documented in @osdi18legoos.

While Memtis appears to demonstrate lower migration overhead in @fig-evaluation-breakdown, this result directly stems from its ineffective hotness classification, which fails to identify sufficient hot pages for migration, a deficiency reflected in its substantially longer overall execution time.
By contrast, Demeter's approach successfully identifies more hot data while maintaining significantly lower migration overhead, delivering the optimal balance between classification accuracy and performance efficiency.

=== Sensitivity Study <h3-sensitivity>
#include "figure/evaluation/sensitivity.typ"

To evaluate Demeter's robustness across diverse parameter configurations, we conducted a sensitivity analysis presented in @fig-evaluation-sensitivity.
This study focuses on key parameters that influence access tracking and hotness classification, two critical components of guest-delegated TMM design.
\
*PEBS Event Parameters.*
Demeter utilizes PEBS with load latency events as detailed in @h3-ept-pebs.
With this design, two primary parameters determine PEBS-based access tracking behavior:
sample frequency and latency threshold.
Sample frequency directly impacts both data quality for hotness classification and overall system overhead.
The latency threshold parameter serves as a filter to distinguish between cache and memory accesses, though setting it too high risks filtering legitimate FMEM samples.

We evaluated these parameters by measuring their effect on the average runtime of nine concurrent virtual machines running the GUPS workload.
For sample frequency, we analyzed its inverse (sample period), which specifies the number of valid memory accesses occurring between consecutive PEBS buffer writes.
As shown in @fig-evaluation-sensitivity, performance remains stable across a wide range of parameter values, with only minor performance degradation observed at extreme choices (very high latency thresholds or large sample periods).
Our chosen values, 4093 for sample period and 64ns for latency threshold, consistently deliver near-optimal performance across diverse workload conditions.
\
*Range Split Parameters.*
Demeter's range-based classification identifies local hotspots when memory ranges accumulate access counts exceeding the split threshold ($tau_"split"$).
The system evaluates potential splits at regular intervals defined by the split period ( $t_"split"$).
We systematically tested various combinations of these parameters to identify optimal configurations and sensitivity boundaries.
The results in @fig-evaluation-sensitivity demonstrate the robust nature of our range-based classification approach.
Significant performance degradation occurs only under extreme parameter settings:
when splits occur too infrequently (split periods exceeding 5000 ms or thresholds above 17) or too frequently (periods below 500ms).
Our selected parameter combination ($tau_"split" = 15, t_"split" = 500"ms"$) consistently delivers optimal performance while providing sufficient responsiveness to changing workload patterns.
This parameter stability significantly simplifies deployment across diverse virtualized environments, as system administrators can confidently use default values without extensive per-workload tuning.


```



#### draft

```
=== Efficient and Scalable Guest-Delegated TMM
#include "figure/evaluation/breakdown.typ"
#include "figure/evaluation/realtime-gups.typ"

*EPT-friendly PEBS Access Tracking.*
To present a low-level performance understanding into our guest-delegated TMM design, we first show a overhead breakdown study in @fig-evaluation-breakdown.
We start nine concurrent running virtual machines with the GUPS workload and record average CPU time spent in various stages of TMM.
Demeter's sample collection methodology through proactive draining during context switches only spent around three seconds in total.
In comparison, Memtis' sample collection worker thread spent 16× more time on average.
Other guest solutions, including TPP and Nomad, relies on walking page tables and checking `PTE.A/D` bits for all managed physical pages, and the best performing one, TPP, spend 1.2× CPU compared to Memtis.
\
*Range-based Hotness Classification.*
To demonstrate the agility and accuracy of our range-based hotness classification design, @fig-evaluation-realtime-gups shows the realtime GUPS throughput results under the same workload as in @fig-evaluation-breakdown.
As TMM identifies hot pages and place them on the FMEM, the instantaneous throughput should rise, where as the maximum throughput value indicates the classification accuracy.
Compared to other guest designs, Demeter shows the steepest slope before 80 seconds, demonstrating its agility in recognizing hot data.
After that, the descending throughput number might due to the migration traffic.
At the end, Demeter is the first to finish with the highest maximum instantaneous GUPS throughput demonstrate accurate identification of the hot data ranges.
The efficiency of Demeter's classification algorithm can be demonstrated by @fig-evaluation-breakdown, we are able to find the most hot data with only $2/3$ overhead compared to the next best alternative.
\
*Balanced Page Relocation.*
Demeter's migration solution despite migrated the most amount of hot data (as indicated by @fig-evaluation-realtime-gups) only consumes around 28% overhead compared to TPP as shown in @fig-evaluation-breakdown, who spend nearly half the time on the software overhead of hinting page fault handler (we do not include hardware faulting overhead due to its complexity @osdi18legoos).
Despite Memtis in @fig-evaluation-breakdown shows the lowest overhead, such result is because the failure in hotness classification, which is shown by the longest overall execution time, yields nearly no data for migration.


=== Sensitivity Study <h3-sensitivity>
#include "figure/evaluation/sensitivity.typ"

To study Demeter's robustness with diverse parameter settings.
We present a sensitivity study in @fig-evaluation-sensitivity.
We mainly explore parameters used during access tracking and hotness classification.
\
*PEBS Event Parameters.*
Demeter use PEBS with load latency event (discussed in @h3-ept-pebs).
In PEBS based access tracking the sample frequency determines the quality of data fed to hotness classification and the overall tracking and classification overhead.
Our load latency event choice expose another latency threshold parameter, which filters out cache accesses, but might also filters out FMEM samples if set too high.
We evaluate these parameters in @fig-evaluation-sensitivity through the effects on the average runtime of nine concurrent virtual machines running the GUPS workload.
For the sample frequency, we evaluate is inverse, i.e. sample period, which determines after how many valid accesses would PMU write a sample to the PEBS buffer.
Results show that, overall the performance variance across parameter settings are relatively small, with minor slowdown when latency threshold or sample period is too large.
And, our choice of 4093 as the sample period and 64 as the latency threshold stay among the best values.
\
*Range Split Parameters.*
Demeter identify local hotspot when ranges have accesses exceeding split threshold $tau_"split"$.
Checks for the split operation are performed after every split period $t_"split"$ seconds has passed.
We evaluate different combinations of these two parameters.
Results in @fig-evaluation-sensitivity again show great robustness of Demeter's range-based hotness classification, severe performance degradation is only found when ranges are split very rare (split period longer than 5 seconds or split threshold larger than 17) or too often (less than 500 ms).
Our parameter choice ($tau_"split" = 15$ and $t_"split" = 500$) demonstrated to be great.



== Real World Applications
#include "figure/evaluation/realworld-workloads.typ"

To demonstrate Demeter's practicality, we select seven representative memory-intensive real-world workloads from databases, scientific computing, graph processing, and machine learning, following Memtis @sosp23memtis, 
For databases, we choose a `btree` database index workload from Mitosis @asplos20mitosis and an in-memory online transactional database engine, `Silo` @sosp13silo.
For scientific computing, we select `bwaves` (603.bwaves_s), a blast wave simulator for fluid dynamics from SPEC CPU 2017 @speccpu17, and `XSBench` @physor14xsbench, a nuclear reactor particle simulator.
We choose `bwaves` due to its scalability and adaptability to different memory sizes.
For graph processing, we include `graph500` @graph500, which uses generated Kronecker graphs, and `PageRank` @arxiv15gapbs applied to a real-world social network graph from Twitter @www10twitter.
Finally, for machine learning, we select multi-core `LibLinear` @jmlr09liblinear, a large-scale linear classification workload with the `kdda` dataset.
We scale up each workload as much as possible without exceeding our test virtual machine’s memory capacity.
And we presents results grouped by application access patterns.
\
*Uniform Access Patterns.*
`btree` and `bwaves` workloads exhibits a relative uniform access patterns.
In `btree`, requests with random keys walks random tree leaves, while in `bwaves`, the simulator iterate through the entire simulation space in rounds.
Despite a relative uniform pattern, Demeter is able to recognize the hotter regions (internal nodes of `btree` and regions around current simulation calculation in `bwaves`) and achieve upto 20% performance speedup when running multiple VMs.
\
*Static Hotspot.*
Demeter shows the most performance improvement in application with static hotspot (i.e., `XSBench` and `LibLinear`) we are able to achieve upto 6.6× time improvement, and comparing to the next best alternative, Memtis, we are able to achieve minimal 40% improvement in `XSBench`.
\
*Dynamic Shifting Hotspot.*
Silo falls into this category, as an OLTP database mainly exhibits temporal locality, with newly inserted data records accessed most often in near future.
In this workload, Demeter is able to achieve a minimal of 10% performance improvement over the second-best alternative, Memtis, and upto 4× performace compared to worst performing one, Nomad.
\
*Skewed Access Pattern.*
Graph processing including `graph500` and `PageRank` often exhibits skewed access pattern due to power-law distribution often found in both real @www10twitter and synthetic @arxiv15gapbs graphs, with the most connected vertices and nodes having the most accesses.
However, due to the small scatter edges in the entire space, Demeter is relatively hard to achieve the best performance.
Overall Demeter is the second best performing one following TPP.


== Hypervisor-based TMM
To compare guest-delegated performance with hypervisor-based solution under their favorable setting, we use TPP with MMU notifier to collect `PTE.A` bits and denote as TPP-H and show results in @fig-evaluation-realworld-workloads.
We run nine virtual machines resulting a total provisioned DRAM of 28.8GiB for guest-based designs, for TPP-H, we boot the machine with 36GiB total DRAM leaving head rooms for hypervisor use.
We leave TPP-H the full freedom to assign any amount of DRAM to any virtual machine as needed.
Results show guest-delegated Demeter provide better overall in six out of seven total workloads with up to 2×, 20% and 10% speedup in dynamic, static hotspot pattern and uniform accesses.
Demeter loses in `PageRank` with 60% slowdown.
Comparing with TPP-H, guest-delegated TPP shows second best overall performance, further demonstrate the advantage of guest-delegation.

== Performance with CXL memory
#include "figure/evaluation/cxl-performance.typ"
To prepare Demeter with emerging CXL memory, we present an evaluation with emulated CXL.mem using remote attached NUMA memory, following Pond @asplos23pond.
CXL.mem exhibits better performance than PMEM and may hide some portion of improvements brought by tiered memory management.
We use the same setting and same set of real world applications as @fig-evaluation-realworld-workloads, subtitling PMEM for emulated CXL.mem and show results in @fig-evaluation-cxl-performance.
The results show Demeter's constant high performance across applications exhibiting static and dynamic shiting hotspots including `Silo`, `LibLinear`, and `XSBench`, with minimal speedup of 10% compared to the next-best alternative, seen in `LibLinear` on TPP.


== Latency Sensitivity Application
#include "figure/evaluation/tail-latency.typ"
To demonstrate Demeter's effect on interactive application that are latency sensitivity, we use the OLTP data base `Silo`.
We follow the same evaluation settings as @fig-evaluation-realworld-workloads and show results in @fig-evaluation-tail-latency.
We report both the average and tail latency percentiles.
For the the 99th percentile, Demeter is able to reduce 4.7% latency compared to the next-performing alternative, TPP.
For the lower percentiles, Demeter is able to be the best performing one or on-par with the second best alternative, with 0.1ms difference.

```



### tmp draft

```

== Understanding Demeter Performance
To understand the performance of Demeter, we present micro-benchmarks demonstrating the elasticity of Demeter's tiered memory provisioning solution Demeter balloon, scalability of Demeter's access tracking solution through guest EPT-friendly PEBS, accuracy and agility of Demeter's range-based hotness classification, and low overhead balanced page relocation.
\
*Workload*
We use the GUPS workload @sosp21hemem as our micro-benchmark workload to evaluate overall system memory throughput behavior.
GUPS measures how many read-modify-write transactions can be performed per second.
We utilize the `hotset` variant of GUPS, which divides the memory space into a hot section and a cold section.
The hot section is accessed ten times more frequently than the cold section, with random memory accesses within each section.

=== Tired Memory Provision
@fig-evaluation-balloon shows that Demeter's TMP solution, Demeter balloon, is able to achieve the same level of performance as static allocation while supporting dynamic tiered-aware memory provision.
We compare Demeter balloon with state-of-the-art memory provision solution, VirtIO balloon @virtio-spec.
Both solutions start the virtual machine with FMEM and SMEM maximal capacity to be 100% the system total memory assignment and rely on balloon resizing to provision the desired FMEM and SMEM allocation ratio ($1/5$ as introduced in @subsec-evaluation-methodology).
Demeter balloon is able to achieve 6× throughput compared to VirtIO balloon.
The reason lies behind VirtIO balloon's unwareness to tiered memory.
It's balloon driver mistakenly reserve FMEM despite the request to adjust SMEM allocation, as a result, FMEM is under-provisioned, causing performance degradation, which greatly contradicts to cloud's elasticity needs.
On the other hand, we compare Demeter balloon with statically assign the desired FMEM ratio during virtual machine boot, we are able to achieve the same level of performance due to correct tiered memory provision.

Demeter balloon's modular design allows it to be used with other kernels, only requiring loading Demeter balloon kernel module.
We add support to state-of-the-art kernel-based tiered memory solutions, including TPP @asplos23tpp and Nimble @asplos19nimble as well as PEBS-based solution Memtis @sosp23memtis.
With Demeter balloon, @fig-evaluation-balloon shows these design are also able to achieve the same level of performance compared to static memory provision.

To highlight Demeter's efficient and scalable guest-delegated tiered memory management apart from the elastic TMP, in the following discussion, we run TMM designs all on top of Demeter balloon.


```

---

### Performance problem with Graph500 and PageRank

Graph500 allocates vertices first and then adjancet matrices, which puts hot data directly on DRAM.

```c
static int64_t maxvtx, nv, sz;
static int64_t * restrict xoff; /* Length 2*nv+2 */
static int64_t * restrict xadjstore; /* Length MINVECT_SIZE + (xoff[nv] == nedge) */
static int64_t * restrict xadj;

int
create_graph_from_edgelist (struct packed_edge *IJ, int64_t nedge)
{
  find_nv (IJ, nedge);
  if (alloc_graph (nedge)) return -1;
  if (setup_deg_off (IJ, nedge)) {
    xfree_large (xoff);
    return -1;
  }
  gather_edges (IJ, nedge);
  return 0;
}
static int
alloc_graph (int64_t nedge)
{
  sz = (2*nv+2) * sizeof (*xoff);
  xoff = xmalloc_large_ext (sz);
  if (!xoff) return -1;
  return 0;
}
```

