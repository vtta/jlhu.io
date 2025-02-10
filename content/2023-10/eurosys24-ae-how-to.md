+++
title = "EuroSys24 AE: How To"
date = "2023-10-09"
# [extra]
# add_toc = true
+++

## How to evaluate the artifacts
- [Evaluator guide](https://sysartifacts.github.io/eurosys2022/guide)

<details>
    <summary>Checklists</summary>

### Artifact Available

- The artifact is available on a public website with a specific version such as a git commit
- The artifact has a “read me” file with a reference to the paper
- Ideally, the artifact should have a license that at least allows use for comparison purposes

Artifacts must meet these criteria *at the time of evaluation*. Promises of future availability, such as artifacts “temporarily” gated behind credentials given to evaluators, are not enough.

### Artifact Functional

- The artifact has a "read me" file with high-level documentation:
  - A description, such as which folders correspond to code, benchmarks, data, …
  - A list of supported environments, including OS, specific hardware if necessary, …
  - Compilation and running instructions, including dependencies and pre-installation steps, with a reasonable degree of automation such as scripts to download and build exotic dependencies
  - Configuration instructions, such as selecting IP addresses or disks
  - Usage instructions, such as analyzing a new data set
  - Instructions for a “minimal working example”
- The artifact has documentation explaining the high-level organization of modules, and code comments explaining non-obvious code, such that other researchers can fully understand it
- The artifact contains all components the paper describes **using the same terminology** as the paper, and **no obsolete code/data**
- If the artifact includes a container/VM, it must also contain a script to create it from scratch

Artifacts must **be usable on other machines** than the authors, though they may require hardware such as specific network cards. Information such as IP addresses must not be hardcoded.

### Results Reproduced

- The artifact has a "read me" file that documents:
  - The **exact** environment the authors used, including OS version and any special hardware
  - The **exact** commands to run to reproduce each claim from the paper
  - The approximate resources used per claim, such as "5 minutes, 1 GB of disk space"
  - The scripts to reproduce claims are documented, allowing researchers to ensure they correspond to the claims; merely producing the right output is not enough
- The artifact's **external dependencies are fetched from well-known sources** such as official websites or GitHub repositories
  - **Changes to such dependencies should be clearly separated**, such as using a patch file or a repository fork with a clear commit history Changes to such dependencies should be clearly separated, such as using a patch file or a repository fork with a clear commit history

</details>

