+++
title = "The Mystery Behind Folio Mapping"
date = "2024-03-22"
# [extra]
# add_toc = true
+++

Each page folio contains a `mapping` pointer,
connecting it back to where the folio belongs.

> Here we only focus on the folios that are in some way accessible to userspace,
> including anonmyous, file and shmem folios.
> Other folio types including slab, page_pool and zone device folios
> are mainly for kernel memory allocation, network stack and special device management.
> They are subsystem implementation specific and has no interest to userspace.
> However, the first three types of folios are important enough
> to be reflected in the design of `struct folio`.
> Only the layout of the first three is directly listed in `struct folio`.
> If you want to work with other types of folios,
> you have to revert back to the old-fashioned "union"-ed `struct page`. 

This mapping pointer is necessary.
For anonmyous folios, we can find which tasks have them mapped.
For file folios, we can know which files (i.e. `inode`s) they belongs to.
And for shmem folios, they can also be used to find the files they belongs to just like file folios.
These similarities and differences can be reflected by the structure the `mapping` pointer points to.
For anonmyous folios, they points to `struct anon_vma`.
And for file and shmem folios, they points to `struct address_space`.
The owning files/`inode`s can be identified by the `host` pointer inside `address_space.
File and shmem `mappings` can further be distinguished by address space operations `a_ops` field.
Shmem folios have `shmem_aops` and file folios have thier filesystem specific `a_ops`,
such as `btrfs_aops`, `ext4_aops`, `f2fs_dblock_aops` and etc.

The `address_space` is pretty straightforward.
What's more interesting is `anon_vma` and how to locate all the tasks or pagetables that map the folio.

## The problem: find all mapping pagetables from a given folio

0. Background: when swap a anonmyous folio to disk,
    the kernel needs to modify all pagetables having the given folio mapped to ensure consistency.
1. Naive solution (`pte_chain`): just add another field
    recording a linked list of all pagetables mapping the folio.
    However, there are two problems:
    1. memory wastage: 8 bytes for every 4KiB memory.
    2. `fork()` performance: when forking a new child process,
        every folio in the parent process has to be modified to record the new child's pagetable.
2. Almost there (naive object based reverse mapping): add an indrect layer.
    For file folio, each folio is owned by a file (hence "object" based reverse mapping).
    So, we can inspect the file via the `mapping` pointer for file folios.
    And for anonmyous folio, each folio is owned by a VMA.
    We can make the `mapping` pointer
    points to an indirect layer called `anon_vma` structure.
    The `anon_vma` has a linked list of VMAs that contain the folio.
    However, there is still a subtle problem:
    - "Phantom" VMA: child process might allocate a new folio
        for a previously CoW mapped parent folio when writing.
        However, the child's VMA would still be linked in the old folio's `anon_vma`.
        Which might result in the VMA linked list contains unrelated VMAs.

<p align="center">
    <img height="200" src="https://static.lwn.net/images/ns/anonvma2.png" alt="anonvma2"></img>
</p>

3. Current solution (object based reverse mapping): add another indirect layer.
    Now, the AV0 (`anon_vma`) has a linked list of AVC0 (`anon_vma_chain`).
    AVC0 then points to the VMA0.
    When forking, new VMA1 is created and its associated AVC1 is also created and linked to the AV0.
    When allocating a new folio for the child due to writing to a CoW folio,
    the old AVC1 is deleted to disconnect the old folio to the child's folio.
    The new folio's AV1 will then be added with a new AVC2 to connect to the child's VMA.

<p align="center">
    <img height="100" src="https://static.lwn.net/images/ns/kernel/avchain2.png" alt=""></img>
    <img height="100" src="https://static.lwn.net/images/ns/kernel/avchain3.png" alt=""></img>
    <img height="100" src="https://static.lwn.net/images/ns/kernel/avchain4.png" alt=""></img>
</p>

> For my opinion, the third solution is too excessive.
> Even [Linus complained about it](https://lwn.net/Articles/383170/).
> The 2nd would be the best considering the clarity and simplicity.
    
### References
- [Linux内核中匿名页的反向映射](https://zhuanlan.zhihu.com/p/573574071)
- [Virtual Memory II: the return of objrmap](https://lwn.net/Articles/75198/)
- [The case of the overly anonymous anon_vma](https://lwn.net/Articles/383162/)
    


## `anon_vma`



