+++
title = "Microbenchmarks"
date = "2023-08-05"
# [extra]
# add_toc = true
+++

## [Plan](../../2023-07/experiment-writing-planning/)
- Collection (kernel PEBS)
    - What is the cost for collecting one access sample?
- Identification (SDH)
    - How accurate is the identified working set compared to ground truth?
- Migration (no special design)
    - Do we really need special migration design?
    - What is the marginal cost?
        - How often does migration actually happen?
        - How much data to migrate each time?


## Collection

### Nimble
Nimble doesn't have its own sample collection technique.
Instead, it relies on Linux's sample source for the active/inactive list.

> we build a holistic multi-level memory solution that directly moves data between heterogeneous memories using the existing OS active/inactive page lists, eliminating another major source of software overhead in current systems.

So, to find out how to measure the cost for collecting one sample,
we first need to determine how Linux **collects** samples for the active/inactive list.

From [lwn](https://lwn.net/Articles/495543/),
we can learn that the cost is mainly the active list accounting:

> The active list contains anonymous and file-backed pages that are thought (by the kernel) to be in active use by some process on the system. The inactive list, instead, contains pages that the kernel thinks might not be in use. When active pages are considered for eviction, they are first moved to the inactive list and unmapped from the address space of the process(es) using them. Thus, once a page moves to the inactive list, any attempt to reference it will generate a page fault; this "soft fault" will cause the page to be moved back to the active list. Pages that sit in the inactive list for long enough are eventually removed from the list and evicted from memory entirely.

From [`folio_add_lru()`](https://elixir.bootlin.com/linux/v6.4/source/mm/swap.c#L501),
we can learn that the accounting process can be broken down to two parts:
- mark the page as active via `folio_mark_accessed()` or wrapper `mark_page_accessed()`
- process a queued batch of pages and finalise the active/inactive decision via `lru_add_drain_cpu()`

> The decision on whether
> to add the page to the [in]active [file|anon] list is deferred until the
> folio_batch is drained. This gives a chance for the caller of `folio_add_lru()`
> have the folio added to the active list using `folio_mark_accessed()`

[Here](https://lpc.events/event/11/contributions/896/attachments/793/1493/slides-r2.pdf)
is a another source for reference.

### Ours
Our samples are all collected through PMI.
We can calculate the total cycle count of all PMI,
then divide it by the number of samples collected.


### HeMem
HeMem's samples are also collected through PMI.
But the major difference is that PMI only sends samples to the kernel.
The kernel then copy them on to a user-visible buffer (perf buffer).
HeMem constantly polls the buffer to find the samples passed from kernel.
So, the cost includes total cycles spent on:
- PMI handling (include copy to perf buffer)
- polling


## Sample Collection

To measure sample collection performance,
we measure average time taken for collecting one sample.
For Nimble and other kernel LRU based design,
this is done via counting how many times `folio_referenced()` is called
and its total time taken via `native_sched_clock()/rdtsc` [^tsc].

We check `folio_referenced()` because every time kernel LRU is checked,
it will inspect each page's hardware access count (`PTE.A`)
on the active and inactive list and update them accordingly.
Check [here](../linux-active-inactive-lists/) for more details on kernel's LRU design.

To count invoke and cycle count we can add additional statistic items
in `vm_event_item` and `vmstat_text`.
Then use `count_vm_event(ITEM)` to make them visiable to userspace via `/proc/vmstat`.


[^tsc]: We can reliably use `rdtsc` because morden Intel processors have a fixed TSC frequency.
You can check this via `TscInvariant` CPU feature, e.g. `sudo cpuid -1 | rg TscInvariant`.
TSC check is also done at kernel boot through `determine_cpu_tsc_frequencies()`.


### Nimble
Nimble's code cannot boot currently, we are trying to rebase it onto a newer version of Linux instead of v5.6-rc6.

| Version  | Status                                                |
| -------- | ----------------------------------------------------- |
| v5.6-rc6 | Boot hang due to virtio related issue                 |
| v5.6     | Boot hang due to NX violation of network drivers      |
| v5.7     | Ditto                                                 |
| v5.8     | Compilation failed due to `arch/x86/entry/thunk_64.o` |
| v5.9     | Ditto                                                 |
| v5.10    | Ditto                                                 |

After trying several candidate versions,
it's obvious old kernel has many unsolved problems.
It's best we can merge Nimble's code into our codebase.

Benefits:
- We gain more insight about Nimble's work, i.e. exactly what gives the most improvement
- Our migration can reuse Nimble's code
- Simpler bechmarking process

