+++
title = "EuroSys24 AE: #53 HDIOV"
date = "2023-10-28"
[extra]
add_toc = true
+++

## #53: HD-IOV: SW-HW Co-designed I/O Virtualization with Scalability and Flexibility for Hyper-Density Cloud

### Paper Summary
This paper proposes an I/O virtualization approach that offers greater scalability and flexibility compared to SR-IOV.
SR-IOV relies on hardware support to expose multiple Virtual Functions (VFs) for
multiplexing PCIe hardware resources across VMs.
However, HD-IOV shifts the emulation of VFs into software,
eliminating the complex management logic in hardware and
increasing the limit on the number of VFs that can be created.
This preserves VMs' direct access to hardware queue pairs ensures optimal performance.
To distinguish between emulated devices,
HD-IOV leverages the IOMMU and uses Process Address Space ID (PASID) as an identifier.
Host CPUs already possess PASID-awareness since VT-d 3.0 on Intel platforms.
However, hardware devices still need to be modified to be compatible with HD-IOV by integrating PASID-awareness.

HD-IOV's host-side device multiplexing logic (VDCM)
is implemented in the Linux kernel and has been upstreamed.
This paper uses the Ubuntu variant of kernel version 5.11.0 on the host.
HD-IOV supports two device models currently, namely NIC and accelerators.
Their guest drivers are implemented in the Linux kernel (CentOS variant of v5.15.4)
and the out-of-tree Intel QAT driver (v0.9.4), respectively.

### Badges
Overall only "Artifact Available" badge is justified.
None of "Functional" and "Reproduced" can be fully justified.
I am unable to identify the exact source files supporting authors design and thus cannot confirm those code are functional.
I am also unable to get a proper evaluation platform to perform required benchmarks and thus I have no support for a reproducable badge either.

Here are the detailed comments:
- ✔️ Available
    - It is justified to award an "Artifact Available" badge to this paper overall.
    All source code in this paper has been either upstreamed to the Linux kernel or merged to vendor drivers.
    They are both publicly available at the time of evaluation.
    However, due to the extensive amount of existing code in both of these two projects,
    it's unclear how to identify which parts correspond to the contributions made in this paper.
- ✖️ Functional
    - It's not well justified to fully award a "Functional" badge.
    Although the source code is publicly available, it is deeply embedded in existing code.
    There is no documentation specifying which directories or files contain the code
    necessary to implement the claimed functionality.
    It is very difficult for future researchers to understand the organization,
    let alone the implementation.
- ✖️ Reproduced
    - This paper does not qualify for a "Reproducible" badge due to several reasons:
    - There is a lack of a proper evaluation platform to fully carry out the evaluation.
    I cannot confirm any one of the for major claims is reproducable.
    Major claims of this paper include performance for both virtualized NIC and accelerator performance.
    Specific NIC (Intel E810) and accelerator (Intel QAT 2.0) hardware are required to perform these benchmarks.
    I am only able to run the accelerator benchmark (figures 14-16) for HD-IOV on a VM with QAT 2.0 provided by the authors.
    However, I cannot reproduce the results for the host nor SR-IOV due to a lack of access to the host machine.
    - There is a lack of proper environment setup scripts to configure the testbed.
    The authors did not provide scripts to set up the required guest network topology
    which is required to perform NIC benchmarks,
    nor did they provide the exact commands to launch the guest VM.
    - There is a lack of automated scripts to reproduce every claim.
    For the NIC benchmark, the reviewers are asked to manually substitute variables,
    such as PKT_SIZE, CLIENT_IP, and so on, in every command used in every subbenchmark.
    - There is no solution for data extraction.
    The accelerator benchmark emits a vast amount of raw output with only a few performance numbers buried in it.
    The authors provided no means for data extraction.
    - There is a mismatch in figures.
    For the results of HD-IOV in figure 15, the throughput numbers match numerically,
    but the unit is operations per second,
    different from the Gbps shown in the figure.

The aboved comments are only temporary.
I am very happy to see any improvements made on the artifact documentation.
I am also very glad to rerun benchmarks on a proper evaluation platform if provided.

### Checklist
### Artifact Available

- ✔️ The artifact is available on a public website with a specific version such as a git commit
- ✔️ The artifact has a “read me” file with a reference to the paper
- ✔️ Ideally, the artifact should have a license that at least allows use for comparison purposes

- ✔️ Artifacts must meet these criteria *at the time of evaluation*. Promises of future availability, such as artifacts “temporarily” gated behind credentials given to evaluators, are not enough.

### Artifact Functional

- ✔️ The artifact has a “read me” file with high-level documentation:
  - ✔️ A description, such as which folders correspond to code, benchmarks, data, …
  - ✔️ A list of supported environments, including OS, specific hardware if necessary, …
  - ✔️ Compilation and running instructions, including dependencies and pre-installation steps, with a reasonable degree of automation such as scripts to download and build exotic dependencies
  - ✖️ Configuration instructions, such as selecting IP addresses or disks
  - ✔️ Usage instructions, such as analyzing a new data set
  - ✔️ Instructions for a “minimal working example”
- ✖️ The artifact has documentation explaining the high-level organization of modules, and code comments explaining non-obvious code, such that other researchers can fully understand it
- ✖️ The artifact contains all components the paper describes using the same terminology as the paper, and no obsolete code/data
- ✖️ If the artifact includes a container/VM, it must also contain a script to create it from scratch

- ✖️ Artifacts must be usable on other machines than the authors’, though they may require hardware such as specific network cards. Information such as IP addresses must not be hardcoded.

### Results Reproduced

- ✔️ The artifact has a “read me” file that documents:
  - ✔️ The exact environment the authors used, including OS version and any special hardware
  - ✖️ The exact commands to run to reproduce each claim from the paper
  - ✖️ The approximate resources used per claim, such as “5 minutes, 1 GB of disk space”
  - ✔️ The scripts to reproduce claims are documented, allowing researchers to ensure they correspond to the claims; merely producing the right output is not enough
- ✖️ The artifact’s external dependencies are fetched from well-known sources such as official websites or GitHub repositories
  - Changes to such dependencies should be clearly separated, such as using a patch file or a repository fork with a clear commit history

- ✖️ The amount of manual work, such as writing configuration files, should be reasonably minimized. In particular, there should be no redundant manual steps such as writing the same configuration values in multiple places, as this inevitably leads to human error.


