+++
+++
下图从左到右分别是guest-hb/guest-vb/host-dram/host-pmem:

<p align="middle">
    <img src="Screenshot 2024-08-13 at 10.47.48.png" alt="vm-hb" style="zoom: 25%;" />
    <img src="Screenshot 2024-08-12 at 09.43.23.png" alt="vm-vb" style="zoom: 20%;" />
    <img src="Screenshot 2024-08-13 at 10.45.18.png" alt="host-dram" style="zoom:25%;" />
    <img src="Screenshot 2024-08-13 at 10.45.36.png" alt="host-pmem" style="zoom:25%;" />
</p>

- vb基本就是PMEM性能
-

However, in our experience at Microsoft Azure, these approaches face severe limitations in virtualized environments (§2). For example, instruction sampling is not supported for VMs and has privacy implications. Fine-grained page table operations consume excessive host CPU cycles [44,79]. In addition, with software-managed tiering, the hypervisor/OS can only manage memory at page granularity. This leads to suboptimal decisions [76] for the common case where a mix of hot and cold data resides on the same page. This is particularly problematic for hypervisors that use larger page sizes (e.g., 2 MB and 1 GB) to reduce overheads. All of these drawbacks make deploying software-based memory tiering techniques unattractive in general-purpose cloud environments.

Memstrata

