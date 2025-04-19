+++
+++

目前我们的环境实际上需要两台机器, 即client以及server. RDMA的pool由server提供, client通过CX7网卡连接并访问. 问题是我们只有一台双路物理机器, 所以考虑将每个node通过DPU做成一个bare meta的VM.

- <https://www.servethehome.com/using-dpus-hands-on-lab-with-the-nvidia-bluefield-2-dpu-and-vmware-vsphere-demo/>
