+++
title = "Bencher Refactor"
date = "2023-08-14"
# [extra]
# add_toc = true
+++

Background: the benchmark framework is too messy to facilitate increasing benchmarking complexity.

Basic requirements for the bencher:
- Argument parsing
- Launch the VM
- Execute the workload

Aim: make the bencher more modular.
There are mainly two types of modules:
- Modules for different variant of kernel
- Modules for different kinds of workload

Put different kernel and workload supporting code into separated modules.
Then abstract away the common workflow among them.

## Argument parsing
Currently all argument parsing is done inside `cli.py`,
additional arguments for each workload has to be specified in the `configure()` function.


## VM lanunching
VM launching needs also to facilitate subtleties acrossing kernels:
- Different commandline for kernel boot;
- Different level of support for `virtiofs`.
    This is the most fundamental requirements to properly run a benchmark.
    Because all required resources are stored on the host and mounted as virtiofs;
- Different kernel module or drivers to be inserted

## Workload
