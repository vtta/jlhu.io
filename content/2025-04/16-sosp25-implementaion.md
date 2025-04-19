+++

+++



#### draft

```


= Implementation
Demeter is implemented based on Linux Kernel v6.10 and Cloud Hypervisor @gh23cloudhypervisor v36.
\
*Hypervisor TMP.*
We implement double ballooning by extending the traditional memory balloon device described in the VirtIO @osr08virtio specification @virtio-spec, repurposing the traditional configuration space and virt-queues for inflation and deflation to control the FMEM sub-balloon.
SMEM sub-balloon is then implemented thourgh mirroring the FMEM sub-balloon's control structures and virt-queues appending to the original configuration space and virt-queue list respectively.
The double ballooning extension is guarded behind an additional tiered memory feature flag, maintaining compability with traiditonal VirtIO balloon drivers.
The Demeter Balloon device and driver are implemented via around 1000 lines of Rust code and 1000 lines of C code respectively.
\
*Guest-delegated TMM.*
Demeter guest is implemented as a standalone Linux kernel module, allowing easily integreting to various kernel versions with minimal modifications.
Demeter guest module is implemented in around additional 8000 lines of C code.

```

