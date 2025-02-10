+++
title = "On-critical-path Migration"
date = "2024-01-12"
# [extra]
# add_toc = true
+++

To reduce the latency between finding a hot page and migrating it to fast memory,
we can try using synchronous migration.
To implement this,
the most intuitive solution is to extend the existing NUMA hinting page faults.
We have known that NUMA faults have the ability to migrate a page during page fault handling.
But the problem is 1) how to trigger NUMA faults for our desired pages and
2) how to ensure the NUMA faults carry out migration instead of only being used to collect access samples.


## NUMA faults triggering

Related code:
```c
// mm/mempolicy.c:639
unsigned long change_prot_numa(struct vm_area_struct *vma,
			unsigned long addr, unsigned long end)

```

## NUMA migration


