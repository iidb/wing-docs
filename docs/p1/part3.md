# Part 3: Tradeoff between range scan and compaction

You should submit your code and report on the Web Learning 网络学堂. In the report, you need to write your answers to the problems.

## Problem 1: lazy leveling (3pts)

We previously implemented the leveling compaction policy: there is only one sorted run in each level, and the size of the latter level is $T$ times the size of the former level. Therefore, the write amplification to compaction one key to the next level is $T$. If there are $L$ levels, the write amplification is $TL$.

Tiering is another compaction policy: there are at most $T$ sorted runs in each level, and when the number of sorted runs in a level reaches $T$, all $T$ sorted runs will be merged (i.e., compacted) into a new sorted run in the next level. Since for each key, it can only be compacted from one level to the next level, a key will be compacted at most $L$ times if there are $L$ levels. Thus the write amplification of the tiering policy is $O(L)$. It is smaller than the write amplification of leveling strategy $O(TL)$. 

However, the tiering policy suffers from high space amplification. In the worst case, there can be $T$ duplicate sorted runs in the last level, and the space amplification can be $T$. It is unacceptable. Thus, lazy leveling is proposed. Lazy leveling combines the advantages of the leveling policy and the tiering policy. Suppose there are $L$ levels in total. Lazy leveling uses the tiering compaction policy in Level $1$, Level $2$, $\cdots$, Level $L-1$ to reduce the write amplification. The maximum number of sorted runs in these levels is $T$. Lazy leveling limits the space amplification by allowing only one sorted run in the last level. When the number of sorted runs in Level $1$, $2$, $\cdots$, $L-2$ reaches $T$, all these sorted runs in the respective level will be merged into a new sorted run in the next level. When the number of sorted runs in Level $L-1$ reaches $T$, all sorted runs in it will be merged into the sorted run in Level $L$.

Here we analyze the write amplification of lazy leveling. Similar to the tiering policy, each non-last level contributes $1$ to the write amplification. The last level uses the leveling compaction policy, therefore it contributes $C$ to the write amplification, where $C$ is the size ratio between Level $L$ and Level $L-1$. Therefore, the total write amplification is $L - 1 + C$. The write cost $w$ is $(L - 1 + C)(input\ size)$.

Your tasks are as follows:

1. Implement lazy leveling in `LazyLevelingCompactionPicker::Get`. You can test it through `test/test_lsm --gtest_filter=LSMTest.LazyLevelingCompactionTest`. Please submit to autolab.
   
2. Measure the write amplification in the test and compare it with the theoretical write amplification $L - 1 + C$ in the report. If they are different, please analyze the reason.

## Problem 2: find the best compaction policy (4pts)

Range scan is a query type that retrieves all records in the given range. To scan a range, we first seek the begin key in all sorted runs, and then sequentially read subsequent records until the end of the range. In problem 2 and problem 3, we only consider short range scan (scan length $\leq 100$). We assume that for each sorted run, we read only one block. So the range scan cost for lazy leveling is $(1 + T (L-1))(block\ size)$, where $1 + T(L-1)$ is the number of sorted runs.

We can generalize the lazy leveling compaction policy by allowing different maximum numbers of sorted runs in non-last levels. Specifically, for Level $i$ where $i < L$, we designate the maximum number of sorted runs as $k_i$. We model the total cost of compactions and range scans in the generalized lazy leveling policy as $f(\vec k, C) = w(\vec k, C) + \alpha r(\vec k)$. The write cost $w(\vec k, C) = (L - 1 + C)(input\ size)$. The read cost $r(\vec k)$ is what you need to model. $\alpha$ describes the workload: a small $\alpha$ for a write-heavy workload and a large $\alpha$ for a scan-heavy workload.

Your task: given the size of the last level $N$, the base level size $F$, the workload parameter $a$, find $\vec k$ and $C$ that minimize $f(\vec k, C)$ and satisfy $N = \prod_{i=1}^{L-1} k_i F C$. You need to design an algorithm to calculate optimal $\vec k, C$ based on parameters. More specifically, you need to implement `FluidCompactionPicker::Get` and adjust the maximum number of sorted runs in each level based on your algorithms. You may explore when and how to adjust the maximum number of sorted runs in each level. For example, you may calculate the optimal $\vec k$ and $C$ for every 5 seconds and apply the changes only when the optimal $\vec k$ or $C$ differs much from the current value. 

We provide a basic benchmark that can be executed by `test/test_lsm --gtest_filter=LSMTest.Part3Benchmark1`. Your algorithm should outperform than the baseline. There is no need to submit to autolab due to the long execution time.

Please write a report detailing the algorithm you have designed and implemented. Additionally, include a comprehensive comparison with other compaction policies. Compare your algorithm with other compaction policies, including leveling, tiering, and lazy leveling in problem 1. Furthermore, adjust the `alpha` value in the benchmark (passing different `alpha` value to `Part3Benchmark` function), and determine the range of `alpha` where your algorithm performs the best. 

The parameters in `Part3Benchmark(alpha, N, scan_length)` are: `alpha` value, `N` is the number of keys, `scan_length` is the length of range scan. `scan_length` is set to be larger than `N` in this problem.

You can get points as long as your solution is reasonable and well-founded.

## Problem 3: find the best compaction policy considering range filters (3pts)

Similar to bloom filters, there are also range filters. They can determine whether keys exist within a specified range in a sorted run. It allows range scans to skip sorted runs without relevant keys, optimizing query performance. We assume that we have a perfect range filter which does not produce false positives. Using this range filter, we can calculate the read cost based on the expected number of sorted runs that need to read. We also assume that the length $m$ of range scan is always the same.

Let the total size of LSM-tree be $S$. Suppose there is a sorted run of size $T$ in the LSM-tree. Assuming that all the keys in the LSM-tree are unique and uniformly distributed, for a range scan of length $m$, the probability that a key from the sorted run will be in the result of the range scan is $1 - \prod_{i=0}^{m-1}\frac{S-T-i}{S-i}$. You can consider for the first key in the result of the range scan, the probability that the key is not in the sorted run is $\frac{S-T}{S}$, and for the second key it is $\frac{S-T-1}{S-1}$, and so on. This probability is approximately $\approx 1-(1-\frac{T}{S})^m\approx 1-e^{-mT/S}$. Then the read cost can be calculated by $r(\vec k)=(\sum_{T} 1-e^{-mT/S})(block\ size)$. The write cost is the same as problem 2.

The code of estimation is in `Part3Benchmark` function.

Your task: given the size of the last level $N$, the base level size $F$, the workload parameter $a$, and the range scan length $m$, approximate the read cost and write cost, find $\vec k, C$ that minimize $f(\vec k, C) = w(\vec k, C) + \alpha r(\vec k)$ and satisfy $N = \prod_{i=1}^{L-1} k_i C F$. You need to implement it in `FluidCompactionPicker::Get`.

We provide a basic benchmark that can be executed by `test/test_lsm --gtest_filter=LSMTest.Part3Benchmark2`. Your algorithm should outperform than the baseline. There is no need to submit to autolab due to the long execution time.

Please write a report detailing the algorithm you have designed and implemented. Additionally, include a comprehensive comparison with other compaction policies. Compare your algorithm with other compaction policies, including leveling, tiering, and lazy leveling in problem 1. Furthermore, adjust the `alpha` value in the benchmark, and determine the range of `alpha` where your algorithm performs the best.

The parameters in `Part3Benchmark(alpha, N, scan_length)` are: `alpha` value, `N` is the number of keys, `scan_length` is the length of range scan. `scan_length` is set to 100 by default. You can also pass different `scan_length` to this function.

You can get points as long as your solution is reasonable and well-founded.
