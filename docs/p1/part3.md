# Part 3: Tradeoff between range scan and compaction

## Problem 1: compaction policy API (4pts)

We previously implemented the leveling compaction policy: there is only one sorted run in each level, and the size of the latter level is $T$ times the size of the former level. Therefore, the write amplification to compaction one key to the next level is $T$. If there are $L$ levels, the write amplification is $TL$.

Tiering is another compaction policy: there are at most $T$ sorted runs in each level, and when the number of sorted runs in a level reaches $T$, all $T$ sorted runs will be merged (i.e., compacted) into a new sorted run in the next level. Since for each key, it can only be compacted from one level to the next level, a key will be compacted at most $L$ times if there are $L$ levels. Thus the write amplification of the tiering policy is $O(L)$. It is smaller than the write amplification of leveling strategy $O(TL)$. 

However, the tiering policy suffers from high space amplification. In the worst case, there can be $T$ duplicate sorted runs in the last level, and the space amplification can be $T$. It is unacceptable. Thus, lazy leveling is proposed. Lazy leveling combines the advantages of the leveling policy and the tiering policy. Lazy leveling limits the space amplification by using the leveling policy in the last level. That is to say, there is only one sorted run in the last level, and data compacted to the last level are merged into the sorted run. Lazy leveling uses the tiering policy in other levels to reduce the write amplification. That is to say, there are $T$ sorted runs in other levels, and data compacted to this level are written as a new sorted run.

Your tasks are as follows:

1. Implement lazy leveling in `LazyLevelingCompactionPicker::Get`. You can test it through `test/test_lsm --gtest_filter=LSMTest.LazyLevelingCompactionTest`.
   
2. Measure the write amplification in the test. Calculate the theoretical write amplification, report the measured result and theoretical result. If they are different, please analyze the reason.

## Problem 2: find the best compaction policy (3pts)

Range scan is a query type that retrieves all records in the given range. To scan a range, we first seek the begin key in all sorted runs, and then sequentially read subsequent records until the end of the range. In problem 2 and problem 3, we only consider short range scan (scan length $\leq 100$). We assume that for each sorted run, we read only one block. So the range scan cost is $r(\vec k)=(1 + \sum_{i=1}^{L-1} k_i)(block\ size)$. 

Then we analyze the write cost. Suppose there are $L$ levels in total. Level $1$, Level $2$, \cdots, Level $L-1$ use the tiering compaction policy, each level contributes $1$ to the write amplification. The last level uses the leveling compaction policy, therefore it contributes $C$ to the write amplification, in which $C$ is the size ratio between Level $L$ and Level $L-1$. Therefore, the total write amplification is $L - 1 + C$. The write cost is $w(\vec k, C) = (L - 1 + C)(input\ size)$. 

We model the total cost of compactions and range scans as $f(\vec k, C) = w(\vec k, C) + a r(\vec k)$, in which $a$ describes the workload: a small $a$ for a write-heavy workload and a large $a$ for a scan-heavy workload.

Your task: given the size of the last level $N$, the base level size $F$, the workload parameter $a$, find $\vec k$ that minimize $f(\vec k, C)$ and satisfy $N = \prod_{i=1}^{L-1} k_i F C$. You need to design an algorithm to calculate optimal $\vec k, C$ based on parameters. More specifically, you need to implement `Part3CompactionPicker::Get` and benchmark it using `TODO`.

Please write a report detailing the algorithm you have designed and implemented. 

You can get points as long as your solution is reasonable and well-founded.

## Problem 3: find the best compaction policy considering range filters (3pts)

Similar to bloom filters, there are also range filters. They can determine whether keys exist within a specified range in a sorted run. It allows range scans to skip sorted runs without relevant keys, optimizing query performance. We assume that we have a perfect range filter which does not produce false positives. Using this range filter, we can calculate the read cost based on the expected number of sorted runs that need to read.

Let the total size of LSM-tree be $S$. Suppose there is a sorted run of size $T$ in the LSM-tree. Assuming that all the keys in the LSM-tree are unique and uniformly distributed, for a range scan of length $m$, the probability that a key from the sorted run will be in the result of the range scan is $1 - \prod_{i=0}^{m-1}\frac{S-T-i}{S-i}$. You can consider for the first key in the result of the range scan, the probability that the key is not in the sorted run is $\frac{S-T}{S}$, and for the second key it is $\frac{S-T-1}{S-1}$, and so on. This probability is approximately $\approx 1-(1-\frac{T}{S})^m\approx 1-e^{-mT/S}$. Then the read cost can be calculated by $r(\vec k)=(\sum_{T} 1-e^{-mT/S})(block\ size)$. The write cost is the same as problem 2.

Your task: given the size of the last level $N$, the base level size $F$, the workload parameter $a$, and the range scan length $m$, approximate the read cost and write cost, find $\vec k, C$ that minimize $f(\vec k, C) = w(\vec k, C) + a r(\vec k)$ and satisfy $N = \prod_{i=1}^{L-1} k_i C F$.

Please write a report detailing the algorithm you have designed and implemented. 

You can get points as long as your solution is reasonable and well-founded.
