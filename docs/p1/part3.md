# Part 3: Tradeoff between range scan and compaction

## Problem 1: compaction policy API (4pts)

We previously implemented the leveling compaction policy: there is only one sorted run in each level, and the size of the latter level is $T$ times the size of the former level. Therefore, the write amplification to compaction one key to the next level is $T$. If there are $L$ levels, the write amplification is $TL$.

Tiering is another compaction policy: there are at most $T$ sorted runs in each level, and when the number of sorted runs in a level reaches $T$, all $T$ sorted runs will be merged (i.e., compacted) into a new sorted run in the next level. During each compaction, a key will be written only once. Therefore, if there are $L$ levels, a key will be compacted at most $L$ times, and the write amplification of the tiering policy is $L$, which is smaller than the write amplification of the leveling policy ($TL$). Although the increased number of sorted runs harms the performance of range scans, the tiering compaction policy is better than the leveling policy if the workload is write-heavy.

However, the tiering policy suffers from high space amplification. There are $T$ sorted runs in the last level, and they may all contain the same set of keys. Therefore, in the worst case, as much as $\frac{T-1}{T}$ of the LSM-tree data are stale data, and only $\frac{1}{T}$ are latest data. The space amplification of the tiering policy can be larger than $T$ in the worst case. It is unacceptable.

Lazy leveling limits the space amplification by allowing only one sorted run in the last level. Lazy leveling allows $T$ sorted runs in other levels to reduce the write amplification.

Your tasks are as follows:

1. Implement lazy leveling. To facilitate the exploration of Problem 2, you need to implement a compaction policy API that you can specify the number of sorted runs in Level $1$ to Level $L-1$. The arguments are $k_1, k_2, \cdots, k_{L-1}, C$, in which $k_i$ stands for the number of sorted runs in Level $i$, and $C$ stands for the size ratio between Level $L$ and Level $L-1$. Note that you should determine the sizes of each level by the size of the last level. For example, if the size of the last level is $N$, the size of Level $L-1$ should be $\frac{N}{C}$, and the size of Level $L-2$ should be $\frac{N}{C k_{L-1}}$.
2. Measure the write amplification in the test `TODO`. Calculate the theoretical write amplification and compare it with the measured one.

## Problem 2: find the best compaction policy (3pts)

Range scan is a query type that retrieves all records in the given range. To scan a range, we first seek the begin key in all sorted runs, and then sequentially read subsequent records until the end of the range. In problem 2 and problem 3, we only consider short range scan (scan length $\leq 100$). We assume that for each sorted run, we read only one block. So the range scan cost is $r(\vec k)=(1 + \sum_{i=1}^{L-1} k_i)(block\ size)$. 

Then we analyze the write cost. Suppose there are $L$ levels in total. Level $1$, Level $2$, \cdots, Level $L-1$ use the tiering compaction policy, each level contributes $1$ to the write amplification. The last level uses the leveling compaction policy, therefore it contributes $C$ to the write amplification, in which $C$ is the size ratio between Level $L$ and Level $L-1$. Therefore, the total write amplification is $L - 1 + C$. The write cost is $w(\vec k, C) = (L - 1 + C)(input\ size)$. 

We model the total cost of compactions and range scans as $f(\vec k, C) = w(\vec k, C) + a r(\vec k)$, in which $a$ describes the workload: a small $a$ for a write-heavy workload and a large $a$ for a scan-heavy workload.

Your task: given the size of the last level $N$, the base level size $F$, the workload parameter $a$, find $\vec k$ that minimize $f(\vec k, C)$ and satisfy $N = \prod_{i=1}^{L-1} k_i F C$. You need to design an algorithm to calculate optimal $\vec k, C$ based on parameters. More specifically, you need to implement `Part3CompactionPicker::Get` and benchmark it using `TODO`

Please write a report detailing the algorithm you have designed and implemented. 

You can get points as long as your solution is reasonable and well-founded.

## Problem 3: find the best compaction policy considering range filters (3pts)

Similar to bloom filters, there are also range filters. They can determine whether keys exist within a specified range in a sorted run. It allows range scans to skip sorted runs without relevant keys, optimizing query performance. We assume that we have a perfect range filter which does not produce false positives. Using this range filter, we can calculate the read cost based on the expected number of sorted runs that need to read.

Let the total size of LSM-tree be $S$. Suppose there is a sorted run of size $T$ in the LSM-tree. Assuming that all the keys in the LSM-tree are unique and uniformly distributed, for a range scan of length $m$, the probability that a key from the sorted run will be in the result of the range scan is $1 - \prod_{i=0}^{m-1}\frac{S-T-i}{S-i}$. You can consider for the first key in the result of the range scan, the probability that the key is not in the sorted run is $\frac{S-T}{S}$, and for the second key it is $\frac{S-T-1}{S-1}$, and so on. This probability is approximately $\approx 1-(1-\frac{T}{S})^m\approx 1-e^{-mT/S}$. Then the read cost can be calculated by $r(\vec k)=(\sum_{T} 1-e^{-mT/S})(block\ size)$. The write cost is the same as problem 2.

Your task: given the size of the last level $N$, the base level size $F$, the workload parameter $a$, and the range scan length $m$, approximate the read cost and write cost, find $\vec k, C$ that minimize $f(\vec k, C) = w(\vec k, C) + a r(\vec k)$ and satisfy $N = \prod_{i=1}^{L-1} k_i C F$.

Please write a report detailing the algorithm you have designed and implemented. 

You can get points as long as your solution is reasonable and well-founded.
