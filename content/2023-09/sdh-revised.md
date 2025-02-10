+++
title = "SDH Revised"
date = "2023-09-07"
# [extra]
# add_toc = true
+++

## Formal definitions and error bound
Similar to the original [paper](https://www.usenix.org/conference/atc18/presentation/gong)
proposing a sketch-like probabilistic top-k algorithm.
We can define our problem like this:

Finding top-$k$ hottest pages refers to finding the most accessed $k$ pages.
Let $\mathscr{A} = \lbrace a_1, a_2, ..., a_N \rbrace$
be a memory access sequence of total $N$ accesses.
Each access $a_j (1 \leq j \leq N)$ hits a page $p_i$,
where $p_i \in \mathscr{P} = \lbrace p_1, p_2, ..., p_M \rbrace$,
and $\mathscr{P}$ is the set of total $M$ pages.
Let $n_i$ be the real access count of page $p_i$ in $\mathscr{A}$.
We order all pages $(p_1, p_2, ..., p_M)$ so that their
access counts are in decending order $(n_1 \geq n_2 \geq ... \geq n_M)$.
The sum of all pages' access count matches $N$, i.e., $N = \sum_{i=0}^M n_i $.

According to the proof from the original [paper](https://www.usenix.org/conference/atc18/presentation/gong),
given a small positive number $\epsilon$, the algorithm achieves
$(\epsilon,\delta) \  counting$ with $\delta = \frac{1}{\epsilon w n_i (b-1)}$, i.e.,
$$P(n_i - \hat{n}_i \geq \lceil \epsilon N \rceil) \leq \frac{1}{\epsilon w n_i (b-1)}$$
$w$ is the width of the bucket array and $P(decay) = b^{-c}, (b \to 1^+)$
is the probability of decaying happens in a bucket with count $c$.


$(\epsilon,\delta) \  counting$ shows that the algorithm achieves a low error rate in estimating the access counts
of the top-$k$ hottest pages.

## Parameter choice
The original implementation has the following empirical equation `4 * W * D + 54 * K <= MEM`,
where `D = 2`, `20KiB ≤ MEM ≤ 100KiB` and `200 ≤ K ≤ 1000`.
From the range of `MEM` we can deduce that `2000 < W < 12000`. 
When giving $b = 1.08, N = 10^7, ε = 2^{-16} \text{or } 2^{-17}$ the error rate is within $(0.01, 0.05)$.

## Empirical requirements 
Samples are collected from either performance monitering interrupt, which is
a kind of NMI, or from the scheduler switching out an old task.
Both situations are performance cirtical, so there should not be any memory
allocations. Because, allocation might trigger an expensive direct reclaim.
Further more, locking is prohibited in NMI context, which also forbids us
from allocating memory.

The auxiliary heap keeps the hottest pages. Those pages could be in two
places, i.e., either DRAM or PMEM. DRAM hot pages are not allowed to be
demoted. The heap in this situation filters out undisireable demotions.
PMEM hot pages serve as promotion candidates.
Because, frequent queries might be made to find out if a page exists in the
heap. There should be an hashtable to serve as an reverse index.
The hashtable maps the address back to the arrray index of heap storage.
Because hashtable's O(1) query/insertion/deletion, the introduction of a
hashtable based reverse index does not change the time complexity.

The special conditions make hand-crafted static heap a necessity.
When swapping elementes in the heap, we need to update the reverse index.
The hashtable will be of a fixed size. As both the heap and hashtable cannot
allocate memory when inserting, which occurs during the above mentioned
performance cirtical context.

