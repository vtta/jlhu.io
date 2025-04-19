+++
+++

MMU notifier到底谁是 event source, 谁是subscriber? 如果按照谁register谁是subscriber的话那么KVM实际上是subscriber. DAMON是一个典型的event source, 他在区间扫描代码中(`damon_young_pmd_entry`属于`__damon_va_check_access`的一部分)会调用MMU notifier. 按照这个理解需要检查A-bit的时候主动调用`mmu_notifier_test_young`. 然后subscriber会被callback, 以KVM为例, KVM这里会主动去扫描EPT里面的A-bit. 所以实际上不需要通过trap来capture guest所有的A-bit置位信息, 只有在收到请求(被callback)时去主动扫描EPT就行了.

这个理解符合reviewer A的comment:

> One notable omission is the lack of evaluation of hypervisor-based tiering. The paper could benefit from experiments comparing guest-based tiering with hypervisor-based tiering. For example, by spawning multiple VMs on a hypervisor running a system like TPP - which could leverage the MMU notifiers to trigger KVM to scan the respective EPT entry of the VM's process. Since guest-based tiering is one of the main contributions of the paper, I believe this experiment is crucial for fully evaluating the claims made in the paper.

我们之前理解的trap guest的PTE修改然后通过MMU notifer通知hotness management实际上是搞反了.

不过我们还是从更重要的角度回复了:

> ... more importantly, host kernel approaches like TPP fail to differentiate between guests. This critical limitation leads to severe memory allocation imbalances, where some guests receive exclusively fast memory while others are relegated to slow memory, undermining fair sharing.

### Details

```c
static int kvm_mmu_notifier_test_young(struct mmu_notifier *mn,
           struct mm_struct *mm,
           unsigned long address)
{
 trace_kvm_test_age_hva(address);

 return kvm_handle_hva_range_no_flush(mn, address, address + 1,
          kvm_test_age_gfn);
}

bool kvm_test_age_gfn(struct kvm *kvm, struct kvm_gfn_range *range)
{
 bool young = false;

 if (kvm_memslots_have_rmaps(kvm))
  young = kvm_handle_gfn_range(kvm, range, kvm_test_age_rmap);

 if (tdp_mmu_enabled)
  young |= kvm_tdp_mmu_test_age_gfn(kvm, range);

 return young;
}

struct kvm_gfn_range {
 struct kvm_memory_slot *slot;
 gfn_t start;
 gfn_t end;
 union kvm_mmu_notifier_arg arg;
 bool may_block;
};

bool kvm_tdp_mmu_test_age_gfn(struct kvm *kvm, struct kvm_gfn_range *range)
{
 return kvm_tdp_mmu_handle_gfn(kvm, range, test_age_gfn);
}

static bool test_age_gfn(struct kvm *kvm, struct tdp_iter *iter,
    struct kvm_gfn_range *range)
{
 return is_accessed_spte(iter->old_spte);
}

static inline bool is_accessed_spte(u64 spte)
{
 u64 accessed_mask = spte_shadow_accessed_mask(spte);

 return accessed_mask ? spte & accessed_mask
        : !is_access_track_spte(spte);
}

static inline u64 spte_shadow_accessed_mask(u64 spte)
{
 KVM_MMU_WARN_ON(!is_shadow_present_pte(spte));
 return spte_ad_enabled(spte) ? shadow_accessed_mask : 0;
}

shadow_accessed_mask = has_ad_bits ? VMX_EPT_ACCESS_BIT : 0ull;

```

可以看到KVM的notification handler中收到的address是hVA. 经由`kvm_handle_hva_range_no_flush`转换成gPA (gfn/guest frame number). 最后再由`kvm_tdp_mmu_handle_gfn`来walk EPT table得到pte. 最后访问entry的Bit 8, 即A-bit.

此外KVM中支持总共三种对A-bit的操作, 即上文的test/`kvm_mmu_notifier_test_young`, 以及clear/`kvm_mmu_notifier_clear_young`和clear+flush/`kvm_mmu_notifier_clear_flush_young`. 其中最后一个除了清除A-bit还会flush TLB (`vmx_flush_tlb_all`). 相关的代码(`mmu_notifier_clear_flush_young`)有被`folio_referenced`使用, 即kernel原生(TPP)的split-LRU管理中会使用.

这也从侧面印证了A-bit相关的hotness management的高开销. 因为1) 手动EPT walk没法享受EPT TLB的加速; 2) 其在消耗每一个A-bit后都需要flush EPT TLB.

DAMON代码中没有使用clear+flush的版本, 实际上是错误的 (Intel要求至少使用INVEPT清除从EPT中缓存的信息, 尽管clear+flush的版本会同时使用INVEPT以及INVVPID). Intel对EPT相关的细节管理可以见[这里](12-ept-details).
