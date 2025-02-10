+++
title = "Streaming Decaying Histogram Revisit"
date = "2023-12-04"
+++

### Accurate access counting
Accurate page hotness tracking relies on accurate memory access book keeping.
The most intuitive approach would be introduce a counter field to the `struct page` page metadata.
However, this cannot be done efficiently.
Currently, the `struct page` structure has a limited size of 64 bytes to be able to fit in a cache line.
It has already been packed with locks, page flags, allocator links, process address space links,
page cache links, DMA metadata and so on.
It is not possible to introduce a wide enough counter to record the page access info
without exceeding the cache line size or introduce unexpected cache misses due to the unalignment.
Prior work ([SOSP23]Memtis) has tried to put the counters in the page table pages trading off the counter access performance.
However, one physical page could be mapped into several address spaces at the same time.
To accurately read the access count for a given page,
one has to find out every page table mapping the given page,
traverse to the page table leaves and sum up every counter to get the final result.

To solve this issue efficiently, we use a sketch-based probabilistic data structure called streaming decaying histogram.
This data structure consists of $d$ pairs of a counter array and a hash function.
Each hash function maps the given page into a slot in the counter array.
Each page is mapped $d$ times to accommodate for hash collusions.
The minimal counter value among all the counters mapped by the hash functions is the estimated access count for the given address.

### Streaming data and counter decaying
As the application runs, new accesses are comming continueously in a streaming manner.
However, application's memory hot spot might shift but the always increasing counter might not be able to capture this.
We introduce a stream decaying mechanism leveraging hash conflicts to ensure past hot spots are reflected by counter decreases.
When finding the minimal among all counters,
if the hashed counter has a fingerprint (a part of the hash value) does not match the hash value of the given address,
we find a conflict.
We will decrease this counter with a probability instead of increasing.

### Migration
Migration consists of two parts, i.e., promotion and demotion.
Promotion aims to migrate hottest pages of memory in the lower tier to the upper tier.
Demotion aims to migrate coldest pages of memory in the upper tier to the lower tier.
Promotion might happen in two cases:
1. Periodically triggered to adapt to the shifting application hot spot;
2. Upper tier size increases when the rebalancer thinks the performance (e.g. upper tier hit ratio) falls behind the expectation.
Demotion might also happen in two cases:
1. Periodically triggered;
2. The performance is over the expectation.

To make sure the overall hottest pages being put in the upper tier,
we should track (a) the coldest pages of the upper tier and (b) hottest page of the lower tier.
We have these options:
1. Maintain a min heap for (a) and a max heap for (b).
    This is possible because we know if one given page is in the upper tier or lower tier.
    And each page's access count has a chance to be increase and decreased.
    The decaying process and promotion will insert upper/promoted tier pages into the min heap.
    Normal access counting and demotion will insert lower/demoted tier pages into the max heap.
    However, this approach is problematic.
    1. The min heap might be emptied very quickly and
        newly promoted pages might have much higer chance of being demoted.
        After a demotion, the min heap will have a gap between its current size and its capacity.
        We cannot compare the access count of the promoted page
        if it is already larger than the max value in the min heap.
        So the promoted page, despite might be more frequently accessed
        than some pages in the upper tier, is inserted to the min heap.
        *Summary: how to make the upper tier decaying (after wich can we do demotion)
        occurs as frequent as the lower tier promotion.*
    2. There might be thrashing
        (a particular set of pages stuck in a continueously promotion-demotion cycle).
        The access counts in the heap might not have an distinct gap.
        To solve this, we might also consider the closet 2's power of the access count.
        If the gap between the two heap is not significant enough, we will skip migration.
        This situation can also serve as the final stable state.
    3. How to determing the heap sizes?
2. Maintain only one global min heap for tracking pages that should be placed at the upper tier.
    This approach is also problematic.
    1. The size would be $O(n)$ and each SDH insertion would cost $O(1) + O(log(n))$ instead of $O(1)$.
    2. Thrashing might still occur. Because during demotion,
        there might not be a significant enough gap between the heap min
        and those pages whose access count are not recored by the heap (i.e., unknown at demotion decision time).
3. Additionally maintain an upper tier and a lower tier page list.
    Because the above 2 options only track a subset of pages,
    no accurate migration decision clould be made once heaps are exhausted.
    This also cannot be done efficiently without introducing additional metadata for a page,
    e.g. store link pointers if we try to organize them into linked lists.
4. Reusing Linux's LRU. The original LRU organize pages into active and inactive lists.
    Given that we can easily know if a page belongs to the upper or the lower tier.
    Thus, we essentially already have a very nice structure: upper inactive list and lower active list.
    However, the traiditonal way to maintain the LRU requires scanning.
    To perform a scan, we need to first isolate (pop from the list tail) pages from a LRU list.
    Then we iterate through the pages and find out if they belong to the active or the inactive list.
    Finally, we can put them back (push to the head) to the respective list.
    We can customize the second step to refill our heaps and perform migration.
    This also gives us an opportunity to offload the migration into a async migration thread.
    The migration thread can perform the scan at a set interval or on woken up when out-of-memory.

### Cooling
As the decaying happens with a probability.
The probability expoentially decreases with the counter value increasing.
To make sure the probability does not become nearly zero,
we can maintain a min counter value for the heap.
The probability can now expoentially decrease with the difference to the min counter.

### Adaptive heap size determination

