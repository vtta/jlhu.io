+++
title = "LRU Page Activation and Deactivation"
date = "2023-12-05"
+++

We know Linux maintains the LRU in a split active-inactive list manner.
However, exactally when would a page be activated (moved to the active list),
deactivated (moved to the inactive list) is still very vague.
Let's try to find out using the source code.

<!-- Conclusion: The activation and deactivation happens  -->

<details><summary>code</summary>

The active and inactive state is indicated by the `active` bit in `page->flags`.
Activation and inactivation cannot escape setting and clearing this bit, via `folio_set_active()` and `folio_clear_active()`.

A quick `rg` finds the following occurances:

| File              | Function                                                     | Purpose                                                      |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `mm/migrate.c`    | `folio_migrate_flags()`                                      | Copy the flags and some other ancillary information during migration |
| `mm/slab.h`       | `slab_set_pfmemalloc()`                                      | Using `active` bit as slab internal management flag          |
| `mm/swap.c`       | `folio_activate_fn()`<br/><=`folio_activate`<br/><=`folio_mark_accessed()` | Activate the page (used as an async callback by `folio_activate()`) |
| `mm/swap.c`       | `__lru_cache_activate_folio()`<br /><= `folio_mark_accessed()` | The folio is already batched for activation but is being marked again, so immediate set the active flag. |
| `mm/swap.c`       | `folio_add_lru()`                                            | (Only when MGLRU is enabled)                                 |
| `mm/vmscan.c`     | `shrink_folio_list()`                                        | A page is found (by `folio_check_references()`) to be mapped by multiple PTE during LRU scanning (the `folio_list` passed in are islolated from LRU list for scanning by `isolate_lru_folios()`). |
| `mm/workingset.c` | `workingset_refault()`                                       |                                                              |

The code of interest here is `folio_add_lru()`.
</details>



