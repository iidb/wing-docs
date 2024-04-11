# Part 3: Tradeoff between range scan and compaction

## Problem 1: compaction policy API (4pts)

We previously implemented the leveling compaction policy: there is only one sorted run in each level, and the size of the latter level is $T$ times the size of the former level. Therefore, the write amplification to compaction one key to the next level is $T$. If there are $L$ levels, the write amplification is $TL$.

Tiering is another compaction policy: there are at most $T$ sorted runs in each level, and when the number of sorted runs in a level reaches $T$, all $T$ sorted runs will be merged (i.e., compacted) into a new sorted run in the next level. During each compaction, a key will be written only once. Therefore, if there are $L$ levels, a key will be compacted at most $L$ times, and the write amplification of the tiering policy is $L$, which is smaller than the write amplification of the leveling policy ($TL$). Although the increased number of sorted runs harms the performance of range scans, the tiering compaction policy is better than the leveling policy if the workload is write-heavy.

However, the tiering policy suffers from high space amplification. There are $T$ sorted runs in the last level, and they may all contain the same set of keys. Therefore, in the worst case, as much as $\frac{T-1}{T}$ of the LSM-tree data are stale data, and only $\frac{1}{T}$ are latest data, i.e., the space amplification of the tiering policy can be larger than $T$ in the worst case, which is unacceptable.

Lazy leveling limits the space amplification by allowing only one sorted run in the last level. Lazy leveling allows $T$ sorted runs in other levels to reduce the write amplification.

Your tasks are as follows:

1. Implement lazy leveling. To facilitate the exploration of Problem 2, you need to implement a compaction policy API that you can specify the number of sorted runs in Level $1$ to Level $L-1$. The arguments are $k_1, k_2, \cdots, k_{L-1}, C$, in which $k_i$ stands for the number of sorted runs in Level $i$, and $C$ stands for the size ratio between Level $L$ and Level $L-1$. Note that you should determine the sizes of each level by the size of the last level. For example, if the size of the last level is $N$, the size of Level $L-1$ should be $\frac{N}{C}$, and the size of Level $L-2$ should be $\frac{N}{C k_{L-1}}$.
2. Measure the write amplification in the test `TODO`. Calculate the theoretical write amplification and compare it with the measured one.

## Problem 2: find the best compaction policy (3pts)

Range scan is a query type that retrieves all records in the given range. To scan a range, we first seek the begin key in all sorted runs, and then sequentially read subsequent records until the end of the range. We don't consider the sequential read cost in the range scan. Therefore, each sorted run contributes equally to the range scan cost, and we can consider that the range scan cost is the number of sorted runs. We denote the range scan cost as $r(\vec k)$, then $r(\vec k) = 1 + \sum_{i=1}^{L-1} k_i$. Although the lazy leveling policy has a smaller write amplification than the leveling policy, it increases the number of sorted runs and harms the performance of range scans.

Then we analyze the write amplification. Level $1$ to Level $L-1$ uses the tiering compaction policy, therefore each level contributes $1$ to the write amplification. The last level uses the leveling compaction policy, therefore it contributes $C$ to the write amplification, in which $C$ is the size ratio between Level $L$ and Level $L-1$. Therefore, the total write amplification $w(\vec k, C) = L - 1 + C$, in which $L-1$ is the length of $\vec k$.

We model the total cost of compactions and range scans as $f(\vec k, C) = w(\vec k, C) + a r(\vec k)$, in which $a$ describes the workload: a small $a$ for a write-heavy workload and a large $a$ for a scan-heavy workload.

Your task: given $a, N, F$, in which $N$ is the size of the last level, $F$ is the write buffer size, find $\vec k$ that minimize $f(\vec k, C)$ and satisfy $N = \prod_{i=1}^{L-1} k_i F C$.

You should verify your solution via experiments.

You can get points as long as your solution is reasonable and well-founded.

## Problem 3: find the best compaction policy considering range filters (3pts)

Range filters can tell whether keys exist within a specified range in a sorted run. This allows range scans to bypass sorted runs without relevant keys, optimizing query performance. We still consider the range scan cost $r$ as the number of sorted runs it reads in this problem. However, since some sorted runs may be skipped, the range scan cost $r \le 1 + \sum_{i=1}^{L-1} k_i$

We denote the range scan length as $m$, then the expected number of records to read (i.e., expected scan length) in the last level (i.e., Level $L$) is $m$. Since the size of Level $L-1$ is $\frac{1}{C}$ of the size of Level $L$, and there are $k_{L-1}$ sorted runs in Level $L-1$, the size of each sorted run in Level $L-1$ is $\frac{1}{C k_{L-1}}$ of the size of Level $L$. Therefore, the expected scan length in each sorted run of Level $L-1$ is $\frac{m}{C k_{L-1}}$. Similarly, the expected scan length in each sorted run of Level $L-2$ is $\frac{m}{C k_{L-1} k_{L-2}}$. To simplify the model, if the expected scan length $s$ of a sorted run $\ge 1$, then we think that the sorted run must be read, and it contributes $1$ to the range scan cost $r$. If $s < 1$, then there is a probability of $1-s$ that the sorted run does not contain keys in the scan range and we don't need to read it, therefore the sorted run contributes $s$ to the range scan cost $r$.

Your task: given $a, N, F$, and the range scan length $m$, find $\vec k, C$ that minimize $f(\vec k, C) = w(\vec k, C) + a r(\vec k)$ and satisfy $N = \prod_{i=1}^{L-1} k_i C F$. The total write amplification $w(\vec k, C) = L - 1 + C$. The range scan cost $r(\vec k)$ is what you need to solve.

You should verify your solution via test $TODO$.

You can get points as long as your solution is reasonable and well-founded.
