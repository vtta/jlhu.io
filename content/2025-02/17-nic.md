最终选择了购买两块BlueField-2 DPU BF2M345A-VENOT_ES[^desc]. 两块总价1700不含运费. 虽然这张卡基于ConnectX-6但是相比ConnectX-7的非DPU型号最低也要2750还是很有性价比的.

另外此卡是QSFP56, 相比ConnectX-7 400GbE 用的OSFP线缆要便宜得多.

[^desc]: `NVIDIA BlueField-2 E-Series Eng. sample DPU; 200GbE single-port QSFP56;`

---

目前已知的坑包括ES版本的固件更新问题. 但是目前[B站](https://www.bilibili.com/video/BV1Cm421s7sq)有更新教程. 大致流程:

- Host侧使用minicom连接DPU的console: `/dev/rshim0/console`

- 向DPU更新[DOCA 1.51-LTS](https://developer.nvidia.com/doca-1-5-1-download-archive)版本.
  - bfb文件下载连接: [`DOCA_1.5.1_BSP_3.9.3_Ubuntu_20.04-4.2211-LTS.signed.bfb`](https://developer.nvidia.com/networking/blue-os-eula?mtag=bluefield_sw_drivers&mrequest=downloads&mtype=BlueField&mver=BFBs&mname=Ubuntu20.04&mfile=DOCA_1.5.1_BSP_3.9.3_Ubuntu_20.04-4.2211-LTS.signed.bfb)
  - 安装DOCA-Host驱动后Host侧执行`bfb-install --bfb DOCA_1.5.1_BSP_3.9.3_Ubuntu_20.04-4.2211-LTS.signed.bfb --rshim rshim0`
  
- DPU侧简单维护.
  - 启动mellanox Software Tools服务: `sudo mst start`.
  - 检查固件版本`sudo mlxfwmanager`: 应为`24.31.0356`. PSID因为`MT_0000000809`
  - 更新DOCA 1.51-LTS自带固件: `sudo /opt/mellanox/mlnx-fw-updater/firmware/mlxfwmanager_sriov_dis_aarch64_41686`
  - 更新成功断电冷重启后检查固件版本`sudo mlxfwmanager`: 应为`24.35.2000`
  
- 向DPU更新[DOCA 2.6](https://developer.nvidia.com/doca-2-6-0-download-archive)版本.
  - bfb文件下载连接: [`DOCA_2.6.0_BSP_4.6.0_Ubuntu_22.04-5.24-01.prod.bfb`](https://developer.nvidia.com/networking/blue-os-eula?mtag=bluefield_sw_drivers&mrequest=downloads&mtype=BlueField&mver=BFBs&mname=Ubuntu22.04&mfile=DOCA_2.6.0_BSP_4.6.0_Ubuntu_22.04-5.24-01.prod.bfb)
  - 执行`bfb-install --bfb DOCA_2.6.0_BSP_4.6.0_Ubuntu_22.04-5.24-01.prod.bfb --rshim rshim0`
  
- DPU侧简单维护
  - DOCA 1.51-LTS后版本自带固件(`/opt/mellanox/mlnx-fw-updater/firmware/`)不再支持这张卡(PSID为`MT_0000000809`)
  - 可以从host侧复制`mlxfwmanager_sriov_dis_aarch64_41686`到DPU侧运行更新固件
  - 此文件可以从DOCA的apt源中的deb包中提取: DOCA-2.9.1的[24.10](https://linux.mellanox.com/public/repo/doca/2.9.1/ubuntu24.04/sbsa-arm64/mlnx-fw-updater_24.10-1.1.4.0_arm64.deb); DOCA-2.10.0的[25.01](https://linux.mellanox.com/public/repo/doca/2.10.0/ubuntu24.04/sbsa-arm64/mlnx-fw-updater_25.01-0.6.0.0_arm64.deb);
  - 已知的DOCA版本可以在[这里](https://developer.nvidia.com/doca-archive)找到
  
- 注意事项
  - 此卡芯片丝印为`P2N982 M42M08T22A1-YDTTEV`与零售版不一致: **不可强刷固件**.
  
    08: 8核; T: E-core; 

---

猜测上述更新流程中更新DOCA 2.6的步骤实际上是想更新到最新版本. 

- 那么是否可以直接刷入目前(2025.02)最新的DOCA 2.9.1 LTS?
  - bfb文件下载连接: [`bf-bundle-2.9.1-40_24.11_ubuntu-22.04_prod.bfb`](https://content.mellanox.com/BlueField/BFBs/Ubuntu22.04/bf-bundle-2.9.1-40_24.11_ubuntu-22.04_prod.bfb)

  - 执行`bfb-install --bfb bf-bundle-2.9.1-40_24.11_ubuntu-22.04_prod.bfb --rshim rshim0`

  - 后续最新固件则可以从DOCA的APT源中获取. DOCA 2.9.1 LTS则对应[这里](https://linux.mellanox.com/public/repo/doca/2.9.1/ubuntu24.04/sbsa-arm64/).

  - 目前看到24.10以及25.01均支持`MT_0000000809`. 考虑直接刷入LTS版的`24.43.2026`固件. 此方法已经先进于[reddit上的进度](https://www.reddit.com/r/nvidia/comments/1davt4l/bluefield2_mbf2m345avenot_es_dpus/).

    ```shell
    $ mlnx-fw-updater_25.01-0.6.0.0_amd64/opt/mellanox/mlnx-fw-updater/firmware/mlxfwmanager_sriov_dis_x86_64_41686 -l | grep 809
    MBF2M345A-VENOT_ES_Ax     MT_0000000809      FW 24.44.1036                --             NVIDIA BlueField-2 E-Series Eng. sample DPU; 200GbE single-port QSFP56; PCIe Gen4 x16; Secure Boot Disabled...
    $ mlnx-fw-updater_24.10-1.1.4.0_amd64/opt/mellanox/mlnx-fw-updater/firmware/mlxfwmanager_sriov_dis_x86_64_41686 -l | grep 809
    MBF2M345A-VENOT_ES_Ax     MT_0000000809      FW 24.43.2026                --             NVIDIA BlueField-2 E-Series Eng. sample DPU; 200GbE single-port QSFP56; PCIe Gen4 x16; Secure Boot Disabled...
    ```

- 实际测试到手是`24.31.0356`先刷DOCA 1.5.2 LTS的[`24.35.3006`](https://linux.mellanox.com/public/repo/doca/1.5.2/ubuntu20.04/aarch64/mlnx-fw-updater_5.8-3.0.5.0.10_arm64.deb)再刷DOCA 2.9.1 LTS的[`24.43.2026`](https://linux.mellanox.com/public/repo/doca/2.9.1/ubuntu24.04/arm64-sbsa/mlnx-fw-updater_24.10-1.1.4.0_arm64.deb)

  - 可以直接刷入2.9.1的bfb然后再升级固件. 
  - 直接升到`24.42.2026`会报错, 需要借助`25.35.3006`中转一下.
  - 注意凡是`mlnx-fw-updater*.deb`名字中有`signed`均不包含`MT_0000000809`.

- 关于协议

  - 此卡似乎只支持RoCE, 没有其他类似型号的InfiniBand HDR支持. 不过[比较](https://cloudswit.ch/blogs/roce-or-infiniband-technical-comparison/)来看对我们的prototyping没有太大差别.

  ```shell
  $ mlnx-fw-updater_24.10-1.1.4.0_amd64/opt/mellanox/mlnx-fw-updater/firmware/mlxfwmanager_sriov_dis_x86_64_41686 -l | grep 345A
  MBF2M345A-HECO_Ax         MT_0000000716      FW 24.43.2026                --             BlueField-2 E-Series DPU; 200GbE/HDR single-port QSFP56; PCIe Gen4 x16; Secure Boot Enabled; Crypto Enabled; 16...
  MBF2M345A-HESO_Ax         MT_0000000715      FW 24.43.2026                --             BlueField-2 E-Series DPU; 200GbE/HDR single-port QSFP56; PCIe Gen4 x16; Secure Boot Enabled; Crypto Disabled; 1...
  MBF2M345A-VENOT_ES_Ax     MT_0000000809      FW 24.43.2026                --             NVIDIA BlueField-2 E-Series Eng. sample DPU; 200GbE single-port QSFP56; PCIe Gen4 x16; Secure Boot Disabled...
  ```

  - 线材选择见[官方文档](https://docs.nvidia.com/networking/display/bluefield2firmwarev24432026lts/validated+and+supported+cables+and+modules#src-3411294832_safe-id-VmFsaWRhdGVkYW5kU3VwcG9ydGVkQ2FibGVzYW5kTW9kdWxlcy1IRFIvMjAwR2JFQ2FibGVz): 比如同时支持IB+EH的MCP1650-H0xxEyy [pdf](https://network.nvidia.com/related-docs/prod_cables/PB_MCP1650-H0xxEyy_200Gbps_QSFP56_DAC.pdf) [web](https://docs.nvidia.com/networking/display/mcp1650h0xxeyyspec) 

    - MCP1650-H002E26 咸鱼目前报价 500
    - 其他版本的官方支持型号在[这里](https://docs.nvidia.com/networking/dpu-doca/index.html#dpu-os)

  ---

  

  - 关于PCIe版本型号后缀的猜测(OCP似乎不用一套规则) [C5](https://network.nvidia.com/files/doc-2020/pb-connectx-5-en-card.pdf) [C6](https://www.macnica.co.jp/business/semiconductor/manufacturers/connectx-6-infiniband-datasheet.pdf) [C7](https://www.nvidia.com/content/dam/en-zz/Solutions/networking/infiniband-adapters/infiniband-connectx7-data-sheet.pdf)

    1. **协议以及速度**

       - InfiniBand [link](https://en.wikipedia.org/wiki/InfiniBand#Performance)
         | EDR   | HDR   | NDR   |
         | ----- | ----- | ----- |
         | 100Gb | 200Gb | 400Gb |

       - Ethernet
         | A     | G     | C      | V      |
         | ----- | ----- | ------ | ------ |
         | 25GbE | 50GbE | 100GbE | 200GbE |

    2. 似乎是中低高端?
       | C    | D    | E    | F    |
       | ---- | ---- | ---- | ---- |
       |      |      |      |      |

    3. 似乎跟特性有关?
       |             | C    | E    | S    | U    | N    |
       | ----------- | ---- | ---- | ---- | ---- | ---- |
       | Crypto      | √    | √    | x    | x    | x    |
       | Secure-boot | ?    | -    | √    | √    | x    |
       | UEFI        |      |      |      | x    |      |

    4. 额外特性?
       | O              | E         |
       | -------------- | --------- |
       | OOB management | SNAP-NVMe |

       



---

散热改造

根据B站视频, 板载有4Pin风扇连接器. 但是未知PWM功能是否可用. 从PCB上看能分辨GND和Vcc. 连接器像是[这个](https://item.szlcsc.com/3172296.html). 外网也有[链接](https://makerselectronics.com/product/jst-smd-right-angle-connector-1-25mm-4-pin). 线缆猜测像是[这个](https://item.szlcsc.com/product/jpg_5919854.html).







