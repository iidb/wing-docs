# Part 3: Tradeoff between range scan and compaction

## Problem 1: compaction policy API (4pts)

We previously implemented the leveling compaction policy: there is only one sorted run in each level, and the size of the latter level is $T$ times to the size of the former level. Therefore, the write amplification to compaction one key the next level is $T$. If there are $L$ levels, the write amplification is $TL$.

Tiering is another compaction policy: there is at most $T$ sorted runs in each level, and when the number of sorted runs in a level reaches $T$, all $T$ sorted runs will be merged (i.e., compacted) into a new sorted run in the next level. During each compaction, a key will be written only once. Therefore, if there are $L$ levels, a key will be compacted at most $L$ times, and the write amplification of the tiering policy is $L$, which is smaller than the write amplification of the leveling policy ($TL$). Although the increased number of sorted runs harms the performance of range scans, the tiering compaction policy is better than the leveling policy if the workload is write-heavy.

However, the tiering policy suffers from high space amplification. There are $T$ sorted runs in the last level, and they may all contain the same set of keys. Therefore, in the worst case, as much as $\frac{T-1}{T}$ of the LSM-tree data are stale data and only $\frac{1}{T}$ are latest data, i.e., the space amplification of the tiering policy can be larger than $T$ in the worst case, which is unacceptable.

Dostoevsky [1] proposes lazy leveling, in which there is only one sorted run in the last level to limit the space amplification and $T$ sorted runs in other levels to reduce the write amplification.

Your tasks are as follows:

1. Implement the compaction policy API. To facilitate the exploration of Problem 2, you need to implement an compaction policy API that you can specify the number of sorted runs in each level. The arguments are $k_1, k_2, \cdots, k_{L-1}$, in which $k_i$ stands for the number of sorted runs in Level $i$.
2. Configure the LSM-tree to use the lazy leveling compaction policy and the leveling compaction policy through the compaction policy API, and then measure their write amplifications.
3. Calcuate the theoritical write amplifications. Compare them with the measured ones.

## Problem 2: find the best compaction policy (6pts)

Although the lazy leveling policy has a smaller write amplification than the leveling policy, it increases the number of sorted runs and harms the performance of range scan.

To simplify the analysis of the cost of range scan, we do not consider range filters. Therefore, each range scan will read every sorted run. We also don't consider the sequential read cost in the range scan. Therefore, each sorted run contributes equally to the range scan cost, and we can consider that the range scan cost is the number of sorted runs. We denote the range scan cost as $r(\vec k)$, then $r(\vec k) = 1 + \sum_{i=1}^{L-1} k_i$.

Then we analysis the write amplification. Level $1$ to Level $L-1$ uses the tiering compaction policy, therefore each level contributes $1$ to the write amplification. The last level uses the leveling compaction policy, therefore it contributes $C$ to the write amplification, in which $C$ is the size ratio between Level $L$ and Level $L-1$. Therefore, the total write amplification $w(\vec k, C) = L - 1 + C$, in which $L-1$ is the length of $\vec k$.

Note that $C$ is originally an argument of the compaction policy in the paper of LSM-bush [2] and requires dynamic level sizes to maintain the size ratios between levels. However, it introduces additional code complexity. Therefore, we don't make $C$ an argument of the compaction policy. Instead, we control the total data size $N = \prod_{i=1}^{L-1} k_i C F$, in which $F$ is the write buffer size, so that the size ratio of the last level is $C$.

We model the total cost of compactions and range scans as $f(\vec k, C) = w(\vec k, C) + a r(\vec k)$, in which $a$ describes the workload: a small $a$ for a write-heavy workload and a large $a$ for a scan-heavy workload.

Your task: given $a, N, F$, find $\vec k, C$ that minimize $f(\vec k, C)$ and satisfy $N = \prod_{i=1}^{L-1} k_i C F$.

You should verify your solution via experiments. You can choose $N, F$ you want in your experiments.

If you can't get the optimal solution, you can still get some points by proposing several reasonable solutions and evaluating them.

## References

[1] Dostoevsky: Better Space-Time Trade-Offs for LSM-Tree Based Key-Value Stores via Adaptive Removal of Superfluous Merging. <https://dl.acm.org/doi/pdf/10.1145/3183713.3196927>

[2] The Log-Structured Merge-Bush & the Wacky Continuum. <https://dl.acm.org/doi/pdf/10.1145/3299869.3319903>
