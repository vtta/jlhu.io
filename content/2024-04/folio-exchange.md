+++
title = "Theory and Implementation behind Folio Exchange"
date = "2024-04-01"
# [extra]
# add_toc = true
+++

## Motivation: why do we need to exchange instead of migration?
In a normal system, migrations work by
first allocating an empty page in the destination NUMA node,
then move the data and metadata into that node.
However, the simple allocation-move scheme no longer works for virtualized environment.

Virtualized environment often overcommits memory,
which leads to huge memory pressure.
This in turns forbid memory allocation to success.
Because the higer the memory pressure, the more judicious the allocator becomes.
Only subsystems that are essential to the system's responsivness are allowed to allocate.
This includes the OOM subsystem, which although consumes some free memory in the works,
but produce a large amount of free memory after killing a macilious process.
However, tiered memory management cannot produce free memory.
Thus, during high memory pressure, we need to rethink how the migration could be done.

The answer is straightforward -- exchange intead of migration.
If we could exchange a promoting folio with a demoting folio,
no intermediate free memory is needed,
allowing tiered memory management to be carried out under huge memory pressure.


## Make it happen

### Scope
Similar to the previous post, we only focus on folios
that are in some way accessible to userspace,
including anonmyous, file (and shmem) folios.

In the first version, we will only implement anonmyous folio exchange.
> TODO:
> - Test anonmyous folio ratio
> - Test the overall improvement
> - Handle ksm correctly

## Existing source code anatomy
The most common migration API in the linux kernel is `migrate_pages()`.
This function is only a retry wrapper around `migrate_pages_batch()`.
```c
migrate_pages()
    => migrate_pages_batch()
    => migrate_pages_sync()
        => migrate_pages_batch()
```

`migrate_pages_batch()` tries to migrate every folio in the linked list passed in.
The migration is broken up into two steps.
Firstly, unmaping all the PTE that maps a particular folio,
so that concurrent access to this folio will be blocked.
Allocation of the destination folio also happens here.
Secondly, move the data and states to the destination folio.
These functions supports both sync and async mode.
In async mode, folios with possible blocking such as locking and IO are skipped.
```c
migrate_pages_batch()
    => migrate_folio_unmap()
    => migrate_folio_move()
        anon  => migrate_folio()
        file  => mapping->a_ops->migrate_folio() / fallback_migrate_folio()
        // shmem =>              =migrate_folio()
```

The actual move step will run most likely three actions.
For file folios, an extra check is carried out to ensure data consistency.
If it contains data modifications,
the migration will be skipped until data are written out to disk.
<!-- The actual migration of a particular folio includes: -->
<!-- 1. Fix the folio's mapping information. -->
```c
migrate_folio()
    => folio_migrate_mapping()
    => folio_copy()
    => folio_migrate_flags()
fallback_migrate_folio()
    dirty => writeout()
    clean => migrate_folio()
```

- `folio_migrate_mapping()`

    This function's name is a bit misleading.
    It's actually used to "replace the page in the mapping".
    It includes replacing the folio's record of which mapping it is in,
    and replacing the mapping's record of which folio is at that location.
    The first one includes changing the folio's `mapping` pointer and
    the index field recording the offset into that mapping.
    The mapping might need to use the private/swap union inside the folio.
    Only the swap case is migrated here,
    the private case is handled by fs-specific migration function.
    The second one includes modifying the mapping's radix-tree,
    changing the folio pointer to the new one.

### fs-specific migration
We know that `migrate_folio_move()` will call different migration function
according to the mapping type of a folio.
These are several commonly seen migration functions:

| Fn                         | mapping | Comment                                                      |
| -------------------------- | ------- | ------------------------------------------------------------ |
| `migrate_folio()`          | `NULL`  | Anonymous memory                                             |
| `buffer_migrate_folio()`   | `ext4`  | Non-DAX (`ext4_da_aops` normal / `ext4_da_aops` delayed allocation mode) |
| `filemap_migrate_folio()`  | `f2fs`  |                                                              |
| `fallback_migrate_folio()` | ?       | Used when fs does not provide one                            |

```c
migrate_folio()
    => folio_migrate_mapping()
    => folio_copy()
    => folio_migrate_flags()
buffer_migrate_folio()
    => buffer_migrate_lock_buffers()
    => folio_migrate_mapping()
    => migrate fs private data
    => folio_copy()
    => folio_migrate_flags()
    => unlock_buffer()
filemap_migrate_folio()
    => folio_migrate_mapping()
    => migrate fs private data
    => folio_copy()
    => folio_migrate_flags()
fallback_migrate_folio()
    => filemap_release_folio()
    => folio_migrate_mapping()
    => folio_copy()
    => folio_migrate_flags()
```

From the above cases, we can formulate a template according to `buffer_migrate_folio()`.
```c
exchange_folio()
    => folio_exchange_fs_prepare()
    => folio_exchange_mapping()
    => folio_exchange_fs_private()
    => folio_exchange_data()
    => folio_exchange_flags()
    => folio_exchange_fs_finish()
```




## Misc

### `tmpfs` and `shmem`
`tmpfs` and `shmem` can be seen as two sets of interface to
read/write memory-backed files and/or communicate between processes via the memory-backed file.
`tmpfs` is a filesystem interface.
One can directly create a file under a `tmpfs` and use `mmap()` to map that file into virtual address space.
`shmem` is a syscall interface.
One can call `shmget()` to create a file and then call `shmat()` to map the file and get a virtual address.
The Linux kernel itself implements shmem using tmpfs.
So, we can treat these two as equal.

References:
- [Linux内核tmpfs/shmem浅析](https://blog.csdn.net/ctthuangcheng/article/details/8916065)

### What is an anon or a file folio exactally?
We know linux separates user-accessible folios into anonmyous and file-backed folios.
But what exactally are they?
What happens when an anon folio swapped into disk?
Is it now an file-backed folio?
What happens to the folios backed by a file in a RAMdisk?
Is it now an anonmyous folio?

> **Anonymous Memory**
>
> The anonymous memory or anonymous mappings represent memory **that is not backed by
> a filesystem.** Such mappings are implicitly created for program’s stack and heap or
> by explicit calls to mmap(2) system call. Usually, the anonymous mappings only
> define virtual memory areas that the program is allowed to access.
> The read accesses will result in creation of a page table entry that references
> a special physical page filled with zeroes. When the program performs a write,
> a regular physical page will be allocated to hold the written data.
> The page will be marked dirty and if the kernel decides to repurpose it,
> the dirty page will be swapped out.

Above is the [official definition for
anonmyous memory](https://docs.kernel.org/admin-guide/mm/concepts.html#anonymous-memory).
For anonmyous folios that are swapped to disk, they are still anonmyous memory.
From [here](https://www.kernel.org/doc/Documentation/vm/pagemap.txt),
we can see such folio might be "SWAPBACKED" or "SWAPCACHE".
Notice that "SWAPCACHE" means there exists a swap entry.
This indicates "SWAPCACHE" folios are swapped into the disk.
But "SWAPBACKED" folios will only potentially be swapped into the disk, but not yet.

For file-backed folios, as long as they are backed by a filesystem,
no matter what kind of filesystem or which kind of media that filesystem uses,
they are file-backed folios.
For the special case of RAMdisk backed folios, the backing filesystem would be tmpfs.
So, they are still file-backed folios.
But the differences will show up during memory reclaimation.
For in-stroage file folios, during reclaimation,
they can be simply write out to disk if dirty or discarded if clean.
But for in-memory file-backed folios, they have to be swapped to disk.
Otherwise, the data will be lost.

In conclusion:
1. anon/file folios are only decided by whether they have a filesystem behind,
    not by what kind or what media that filesystem is/uses.
2. Both anon/file folio could be in the swap (or called swapcache).
    If a file folio is in the swap, we know the backing filesystem must be tmpfs (for now).
3. When we say shmem folio,
    we are actually refering to a file folio with tmpfs as the backing filesystem.

