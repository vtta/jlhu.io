+++
title = "EuroSys24 AE: #33 Puddle"
date = "2023-10-30"
[extra]
add_toc = true
+++

## #33: Puddles: Application-Independent Recovery and Location-Independent Data for Persistent Memory

### Paper summary
This paper propose a PMDK-like shareable, migratable, consistent and backwards compatibale PMEM abstraction.
At the core of this abstraction is a puddle, which is a location-independent persistent memory chunk.
The location-independency is realized by 
tracking every backwards compatibale native pointer in every object at object allocation time,
rewriting them at puddle mapping time, and mapping additional puddles pointed by them.
Any uncaught pointers are rewriten at page fault time through user space fault handler enabled by userfaultfd.
The consistent recovery of puddles are enabled through PMDK-like transactional API and
realized through centralized logging and replaying managed by the system puddled daemon.

### Badge
The artifacts is complete, publicly available, well organized and easy to understand.
I am able to reproduce all major claims (figure 9-11) with one click on the author provided docker image.
My only concern is that half the paper is discussing the promising feature of relocatability.
But the related evaluation (figure 13-14) is not included in major claims and also not repoducable.

Overall, it's fair to award all available, functional and reproduced badge to this artifact.
It's nice to see artifacts as complete as this and it will cultivate a positive environment in the research community.


--- 
Improved version by ChatGPT:

## #33: Puddles: Application-Independent Recovery and Location-Independent Data for Persistent Memory

### Paper Summary
This paper proposes a PMDK-like, shareable, migratable, consistent, and backwards-compatible PMEM (Persistent Memory) abstraction.
At the core of this abstraction is a "puddle," which is a location-independent persistent memory chunk.
The location-independency is achieved by
tracking every backwards-compatible native pointer in every object at the time of object allocation,
rewriting them during puddle mapping, and cascaded mapping additional puddles pointed to by them.
Any uncaught pointers are rewritten at page fault time through a user space fault handler enabled by userfaultfd.
The consistent recovery of puddles is enabled through a PMDK-like transactional API and
is realized through centralized logging and replay managed by the system puddled daemon.

### Badge
The artifact is complete, publicly available, well-organized, and easy to understand.
I am able to reproduce all major claims (figure 9-11) with a single click on the author-provided Docker image.
My only concern is that half of the paper discusses the promising feature of relocatability.
However, the related evaluation (figure 13-14) is not included in the major claims and is also not reproducible.

Overall, it's fair to award all available, functional, and reproduced badges to this artifact.
It's great to see artifacts as complete as this, and it will foster a positive environment in the research community.

### Checklist
Artifact Available
- ✔️ The artifact is available on a public website with a specific version such as a git commit
- ✔️ The artifact has a “read me” file with a reference to the paper
- ✔️ Ideally, the artifact should have a license that at least allows use for comparison purposes

- ✔️ Artifacts must meet these criteria *at the time of evaluation*. Promises of future availability, such as artifacts “temporarily” gated behind credentials given to evaluators, are not enough.

Artifact Functional
- ✔️ The artifact has a “read me” file with high-level documentation:
  - ✔️ A description, such as which folders correspond to code, benchmarks, data, …
  - ✔️ A list of supported environments, including OS, specific hardware if necessary, …
  - ✔️ Compilation and running instructions, including dependencies and pre-installation steps, with a reasonable degree of automation such as scripts to download and build exotic dependencies
  - ✔️ Configuration instructions, such as selecting IP addresses or disks
  - ✔️ Usage instructions, such as analyzing a new data set
  - ✔️ Instructions for a “minimal working example”
- ✔️ The artifact has documentation explaining the high-level organization of modules, and code comments explaining non-obvious code, such that other researchers can fully understand it
- ✔️ The artifact contains all components the paper describes using the same terminology as the paper, and no obsolete code/data
- ✔️ If the artifact includes a container/VM, it must also contain a script to create it from scratch

- ✔️ Artifacts must be usable on other machines than the authors’, though they may require hardware such as specific network cards. Information such as IP addresses must not be hardcoded.

Results Reproduced
- ✔️ The artifact has a “read me” file that documents:
  - ✔️ The exact environment the authors used, including OS version and any special hardware
  - ✔️ The exact commands to run to reproduce each claim from the paper
  - ✖️ The approximate resources used per claim, such as “5 minutes, 1 GB of disk space”
  - ✔️ The scripts to reproduce claims are documented, allowing researchers to ensure they correspond to the claims; merely producing the right output is not enough
- ✔️ The artifact’s external dependencies are fetched from well-known sources such as official websites or GitHub repositories
  - Changes to such dependencies should be clearly separated, such as using a patch file or a repository fork with a clear commit history

- ✔️ The amount of manual work, such as writing configuration files, should be reasonably minimized. In particular, there should be no redundant manual steps such as writing the same configuration values in multiple places, as this inevitably leads to human error.

