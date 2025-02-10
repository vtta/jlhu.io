+++
title = "Why Should We Switch to Firecracker?"
date = "2023-11-20"

+++

***THE CONCLUSION is NOT SWITCHING?***



- More supportive scripts available ~~*We can generate such things, e.g.  API client generation from cloudhypervisor's OpenAPI 3.0 specification*~~ (The [generator](https://github.com/openapi-generators/openapi-python-client) does not understand the VM-related concepts embedded in the API yaml)
- More documentation available
- Simplicity: less device supported, simpler emulation logic (e.g. virtio-mmio)
- Finer control on CPUID

=> Do these make building host rebalancer easier?



## Rebalancer logic

Gather all memory accesses and balloon info for every VM at a certain interval. Check if the desired proportion of accesses hit DRAM. 



## Firecracker architecture

- Thread model: 1 Firecracker process <=> 1 microVM. The process runs API, VMM and vCPU(s) threads.
- 

## Firecracker problems

NO GUEST NUMA SUPPORT
