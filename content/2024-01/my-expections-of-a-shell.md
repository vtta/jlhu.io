+++
title = "My Expectations of a Shell"
date = "2024-01-18"
# [extra]
# add_toc = true
+++

Yesterday, I was improving my kernel building script.
Painfully, I have to use arguement parsing again because of the increasing complexity.
We have a bash built-in `getopts` which cannot handle long argument name.
For that we need to switch to a standalone utility `getopt`.
We then have to use an ugly `switch-case` with double semicolon.

I wish I had something like rust's `clap` or python's `fire`.
Easy argument parsing should be a basic feature of a shell!

That would be one of my expectations of a shell.
Others include:
- Easy to execute any commands
- Have support for both string manipulations and arithmethic operations
