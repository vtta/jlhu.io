+++
title = "Nimble Kernel Modifications"
date = "2023-08-16"
# [extra]
# add_toc = true
+++

Nimble's implementation mainly focus on enhancing page migration.
It also provides a interface to directly interact with this via a new system call.

### Involed files
<details>
  <summary>Diff summary compared to v5.6-rc6</summary>

```diff
# git diff --compact-summary fb33c65 8193cabe
 README (gone)                          |   18 -
 README.md (new)                        |   52 +
 arch/x86/entry/syscalls/syscall_64.tbl |    2 +
 fs/aio.c                               |    4 +-
 fs/f2fs/data.c                         |    6 +-
 fs/hugetlbfs/inode.c                   |    4 +-
 fs/iomap/buffered-io.c                 |    4 +-
 fs/proc/base.c                         |  178 ++-
 fs/ubifs/file.c                        |    4 +-
 include/linux/cgroup-defs.h            |    1 +
 include/linux/exchange.h (new)         |   27 +
 include/linux/highmem.h                |    2 +
 include/linux/huge_mm.h                |    5 +-
 include/linux/ksm.h                    |    7 +
 include/linux/memcontrol.h             |   75 +-
 include/linux/migrate.h                |   12 +-
 include/linux/migrate_mode.h           |   10 +-
 include/linux/mm_inline.h              |   21 +
 include/linux/sched.h                  |   46 +
 include/linux/sched/coredump.h         |    1 +
 include/linux/sched/signal.h           |    1 +
 include/linux/sched/sysctl.h           |    3 +
 include/linux/syscalls.h               |   10 +
 include/uapi/linux/mempolicy.h         |    9 +-
 kernel/exit.c                          |   31 +
 kernel/fork.c                          |    7 +
 kernel/sysctl.c                        |   97 +-
 mm/Kconfig                             |   15 +
 mm/Makefile                            |    6 +
 mm/balloon_compaction.c                |    2 +-
 mm/compaction.c                        |   44 +-
 mm/copy_page.c (new)                   |  706 ++++++++++
 mm/exchange.c (new)                    | 1952 +++++++++++++++++++++++++++
 mm/exchange_page.c (new)               |  226 ++++
 mm/huge_memory.c                       |    1 +
 mm/internal.h                          |   22 +
 mm/ksm.c                               |   35 +
 mm/memcontrol.c                        |   82 +-
 mm/memory_manage.c (new)               |  935 +++++++++++++
 mm/mempolicy.c                         |   38 +-
 mm/migrate.c                           | 1006 +++++++++++++-
 mm/vmscan.c                            |   24 +-
 mm/zsmalloc.c                          |    2 +-
 43 files changed, 5593 insertions(+), 140 deletions(-)
```
</details>

[full patch](../nimble-kernel-modifications-nimble-full.patch)

Most of the changes reside in a few newly added files:
- `mm/exchange.c`
- `mm/memory_manage.c`
- `mm/copy_page.c`
- `mm/exchange_page.c`

And existing memory migration source code:
- `mm/migrate.c`

Minor changes are also made to export profiling information to the userspace in `procfs`:
- `fs/proc/base.c`

Other details:

<details>
  <summary>Diff without above mentioned files</summary>

```diff
# echo mm/{exchange,memory_manage,copy_page,exchange_page,migrate}.c fs/proc/base.c | xargs -n1 | xargs -I{} echo ':!{}' | xargs git diff --compact-summary fb33c65 8193cabe -- .
 README (gone)                          | 18 ------
 README.md (new)                        | 52 ++++++++++++++++
 arch/x86/entry/syscalls/syscall_64.tbl |  2 +
 fs/aio.c                               |  4 +-
 fs/f2fs/data.c                         |  6 +-
 fs/hugetlbfs/inode.c                   |  4 +-
 fs/iomap/buffered-io.c                 |  4 +-
 fs/ubifs/file.c                        |  4 +-
 include/linux/cgroup-defs.h            |  1 +
 include/linux/exchange.h (new)         | 27 ++++++++
 include/linux/highmem.h                |  2 +
 include/linux/huge_mm.h                |  5 +-
 include/linux/ksm.h                    |  7 +++
 include/linux/memcontrol.h             | 75 +++++++++++++++++++++-
 include/linux/migrate.h                | 12 +++-
 include/linux/migrate_mode.h           | 10 ++-
 include/linux/mm_inline.h              | 21 +++++++
 include/linux/sched.h                  | 46 ++++++++++++++
 include/linux/sched/coredump.h         |  1 +
 include/linux/sched/signal.h           |  1 +
 include/linux/sched/sysctl.h           |  3 +
 include/linux/syscalls.h               | 10 +++
 include/uapi/linux/mempolicy.h         |  9 ++-
 kernel/exit.c                          | 31 +++++++++
 kernel/fork.c                          |  7 +++
 kernel/sysctl.c                        | 97 ++++++++++++++++++++++++++---
 mm/Kconfig                             | 15 +++++
 mm/Makefile                            |  6 ++
 mm/balloon_compaction.c                |  2 +-
 mm/compaction.c                        | 44 ++++++++-----
 mm/huge_memory.c                       |  1 +
 mm/internal.h                          | 22 +++++++
 mm/ksm.c                               | 35 +++++++++++
 mm/memcontrol.c                        | 82 +++++++++++++++++++++++-
 mm/mempolicy.c                         | 38 ++++++++++-
 mm/vmscan.c                            | 24 +------
 mm/zsmalloc.c                          |  2 +-
 37 files changed, 642 insertions(+), 88 deletions(-)
```
</details>

[major](../nimble-kernel-modifications-nimble-split-major.patch)
and
[minor](../nimble-kernel-modifications-nimble-split-minor.patch)
patch

### Page migration
- Added new migration modes in `enum migrate_mode`
- Added function `migrate_pages_concur()`

### New system call
- `sys_exchange_pages`
- `sys_mm_manage`
- Page allocation time manage in `alloc_pages_vma()`

### Profiling
- Helper functions in `memcontrol.h` to read sizes
- Migration statistics and breakdown definitions in `sched.h`
- Migration statistics and breakdown handling in `exit.c` and `migrate.c`
- `sysctl` knobs in `sysctl.c`

