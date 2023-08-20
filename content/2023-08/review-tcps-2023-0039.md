+++
title = "Review: [TCPS '23] A Fault Resilient Storage Architecture for Cyber-Physical Systems"
date = "2023-08-20"
# [extra]
# add_toc = true
+++

### What the heck is a Cyber-Physical System?
[Wikipedia](https://en.wikipedia.org/wiki/Cyber-physical_system) says:
> Unlike more traditional embedded systems,
> a full-fledged CPS is typically designed as
> a network of interacting elements with
> physical input and output instead of as standalone devices.

> Examples of CPS include smart grid,
> autonomous automobile systems, medical monitoring,
> industrial control systems, robotics systems,
> recycling and automatic pilot avionics.

So basicly, CPS is a fancier term for IoT.
The emphasis is on *"cyber"*, i.e. a netwrok of devices as a whole.


### What is the key problem this paper trying to address?
How to achieve fault-tolerant and high-performance in the mean time in a storage system.





## Comments
### Summary
This paper tries to address the fault-tolerance and performance issue in a distributed storage system.
Fault-tolerance is achieved through replicating data items to multiple nodes,
with each node given by consistent hashing.
High-performance is achieved by saving disk IO in lookups via a in-memory bloom-filter.

However, it's unclear what are the differences in the problem considered 
or the technique used compared to the state-of-the-art. 
This paper also does not include any experimental evaluation comparing the state-of-the-art.



### Comments
- It's unclear what are the main contributions.
- It's unclear what is different between CPS storage considered by the authors
    and well-studied distributed storage systems.
- It's unclear what is different between the proposed consistent hashing and
    those in the literature, such as the one in the [dynamo](https://doi.org/10.1145/1294261.1294281) paper.
- This paper does not show any performance comparison with the state-of-the-art.






