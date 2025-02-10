+++
title = "KVM: vPMU"
date = "2023-12-12"
[extra]
add_toc = true
+++

## PMU architecture support
We have the following registers:

| Register                                | Address     | Purpose                                      |
| --------------------------------------- | ----------- | -------------------------------------------- |
| `PMC0..PMCm`                            | `0xc1 + i`  |                                              |
| `PERFEVTSEL0..PERFEVTSELm`              | `0x186 + i` | Configure which event `PMCi` counts          |
| `FIXED_PMC0..FIXED_PMCn`                | `0x309 + i` |                                              |
| `FIXED_CTR_CTL`                         | `0x38d`     | Configure which events all `FIXED_PMC` count |
| `PERF_CAPACILITIES`                     | `0x345`     | (Read-only) Indicating perfmon version       |
| `GLOBAL_STATUS`                         | `0x38e`     |                                              |
| `GLOBAL_CTRL`                           | `0x38f`     |                                              |
| `GLOBAL_STATUS_RESET`/`GLOBAL_OVF_CTRL` | `0x390`     |                                              |
| `GLOBAL_STATUS_SET`                     | `0x391`     |                                              |
| `GLOBAL_INUSE`                          | `0x392`     |                                              |
| `PEBS_ENABLE`                           | `0x3f1`     |                                              |
| `PEBS_DATA_CFG`                         | `0x3f2`     |                                              |
| `DS_AREA`                               | `0x600`     |                                              |

In theory, there could be a total of 64 PMU counters,
which consisting of 32 general purpose counters and 32 fixed purpose counters.
But in reality, our Icelake 8360Y has 8 general and 4 fixed counters per logical processor. 

This can be reflected by the design of some control registers,
such as `GLOBAL_CTRL` which is 64 bits long.
`GLOBAL_CTRL`'s lower bits are used for enabling/disabling general purpose counters,
and the bits starting at the 32-nd bit are for fixed purpose counters.


I have made a [cheatsheet](../../perf-msrs/pebs-msrs.png):
<details><summary>cheatsheet</summary>

![pmu-msrs](../../perf-msrs/pebs-msrs.png)
</details>


## Linux perf core

The job of Linux perf core is to facilite the sharing of the PMU hardware.
It should be able to let each user process think it owns the PMU exclusively,
and perform state saving or switching seamlessly.

### Concepts
The initialization code `intel_pmi_init()` of Linux perf core will emit such dmesg on kernel boot:
```
[    0.709089] Performance Events: PEBS fmt4+-baseline,  AnyThread deprecated, Icelake events, 32-deep LBR, full-width counters, Intel PMU driver.
[    0.709089] ... version:                5
[    0.709089] ... bit width:              48
[    0.709089] ... generic registers:      8
[    0.709089] ... value mask:             0000ffffffffffff
[    0.709089] ... max period:             00007fffffffffff
[    0.709089] ... fixed-purpose events:   4
[    0.709089] ... event mask:             0001000f000000ff

```

| Name    | Meaning                       | Where       |
| ------- | ----------------------------- | ----------- |
| Version | Architectural perfmon version | `cupid(10)` |
| Format  | PEBS record format            |             |



### Data structures
- `PERF_CAPACILITIES`
    ```c
    union perf_capabilities {
        struct {
            u64	lbr_format:6;
            // control whether the sample is either (fault-like) (=0) taken before the sampled instruction or (trap-like) (=1) after the next event completion.
            u64	pebs_trap:1;
            u64	pebs_arch_reg:1;
            u64	pebs_format:4;
            u64	smm_freeze:1;
            /*
             * PMU supports separate counter range for writing
             * values > 32bit.
             */
            u64	full_width_write:1;
            u64     pebs_baseline:1;
            u64	perf_metrics:1;
            u64	pebs_output_pt_available:1;
            u64	pebs_timing_info:1;
            u64	anythread_deprecated:1;
        };
        u64	capabilities;
    };
    ```

### Global initialization

### Event creation

### PEBS buffer draining


## KVM vPMU architecture

KVM will initialize vPMU in `kvm_init_pmu_capability()` and `kvm_ops_update()->kvm_pmu_ops_update()`.
`kvm_pmu_ops_update()` will update ops according to `intel_pmu_ops` listed below:
```c
struct kvm_pmu_ops intel_pmu_ops __initdata = {
	.hw_event_available = intel_hw_event_available,
	.pmc_idx_to_pmc = intel_pmc_idx_to_pmc,
	.rdpmc_ecx_to_pmc = intel_rdpmc_ecx_to_pmc,
	.msr_idx_to_pmc = intel_msr_idx_to_pmc,
	.is_valid_rdpmc_ecx = intel_is_valid_rdpmc_ecx,
	.is_valid_msr = intel_is_valid_msr,
	.get_msr = intel_pmu_get_msr,
	.set_msr = intel_pmu_set_msr,
	.refresh = intel_pmu_refresh,
	.init = intel_pmu_init,
	.reset = intel_pmu_reset,
	.deliver_pmi = intel_pmu_deliver_pmi,
	.cleanup = intel_pmu_cleanup,
	.EVENTSEL_EVENT = ARCH_PERFMON_EVENTSEL_EVENT,
	.MAX_NR_GP_COUNTERS = KVM_INTEL_PMC_MAX_GENERIC,
	.MIN_NR_GP_COUNTERS = 1,
};
```
These `ops` could be called via `static_call(kvm_x86_pmu_<OP>)(<ARGS>)`,
e.g. `static_call(kvm_x86_pmu_get_msr)(vcpu, msr_info)`.


## Linux perf core in guests

This is what the the same perf core initialization code in guests emits:
```
[    0.499971] Performance Events: PEBS fmt4+-baseline, Icelake events, 32-deep LBR, full-width counters, Intel PMU driver.
[    0.500059] ... version:                2
[    0.500061] ... bit width:              48
[    0.500062] ... generic registers:      8
[    0.500063] ... value mask:             0000ffffffffffff
[    0.500064] ... max period:             00007fffffffffff
[    0.500064] ... fixed-purpose events:   3
[    0.500065] ... event mask:             00000007000000ff
```

