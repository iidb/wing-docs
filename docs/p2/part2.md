# Part 2

In part 2, you will implement a bottom-up optimizer. You will implement the DP algorithm to calculate the optimal join order.

## Add statistics in your executor

You need to add the following code in your join executor and hash join executor:

```cpp
virtual size_t GetTotalOutputSize() const override {
  return ch0_->GetTotalOutputSize() + ch1_->GetTotalOutputSize() +
          stat_output_size_;
}
```

## DP algorithm

We only consider about optimizing the query as: (1) The root executor is a project executor. There is no other project executor in the executor tree. (2) The descendants of the root are join/hash join executors, except for leaves. (3) The leaf executors are sequential scan executors. The SQL statements like this: `select <columns> from <tables> where <predicates>`. For example, `select * from A, B, C where A.id = B.id and B.id = C.id;` is such a query. `select max(a) from A, B where A.id = B.id;` is not, because it has an aggregate executor. `select * from (select * from A), B;` is not, because it has multiple project executors.

The DP algorithm is:

Let $f(S)$ be the cost of joining tables in set $S$. Then we have $f(S)=\min_{T\in S, T\neq \emptyset, S} cost(T, S-T)+f(T)+f(S-T)$ where $cost(T, S-T)$ is the cost of joining $T$ and $S-T$. We assume that the number of tables is small and you can use bits to represent the existence of tables in $S$. For example, if there are 5 tables `A, B, C, D, E`, then $S = 10$ represents $\{B, D\}$. You can use the following method to enumerate all subsets of $S$ so that the time complexity is $O(3^n)$, where $n$ is the number of tables:

```cpp
// T is the subset.
for (int T = (S - 1) & S; T != 0; T = (T - 1) & S) {
  // ...
}
```

## Cost function

You can find `nested_loop_join_cost` and `hash_join_cost` in the `OptimizerOptions`. 

The cost of nested loop join is: **nested_loop_join_cost** $\times$ (**build table size**) $\times$ (**probe table size**)

The cost of hash join is: **hash_join_cost** $\times$ ((**build table size**) + (**probe table size**))

We provide the true cardinality for each table set. It is stored in `true_cardinality_hints` in the `OptimizerOptions`.

## Test

TODO.
