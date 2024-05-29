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

We are only considering optimizing the query under the following conditions: (1) The root executor is a project executor, with no other project executor in the executor tree. (2) The descendants of the root are join/hash join executors, except for leaves. (3) The leaf executors are sequential scan executors. (4) The number of tables is small. The SQL statements like this: `select <columns> from <tables> where <predicates>`. For example, `select * from A, B, C where A.id = B.id and B.id = C.id;` is such a query. `select max(a) from A, B where A.id = B.id;` is not, because it has an aggregate executor. `select * from (select * from A), B;` is not, because it has multiple project executors.

The DP algorithm is:

Let $f(S)$ be the cost of joining tables in set $S$. Then we have $f(S)=\min_{T\in S, T\neq \emptyset, S} cost(T, S-T)+f(T)+f(S-T)$ where $cost(T, S-T)$ is the cost of joining $T$ and $S-T$. We assume that the number of tables is small and you can use bits to represent the existence of tables in $S$. For example, if there are 5 tables `A, B, C, D, E`, then $S = 10$ represents $\{B, D\}$. You can use the following method to enumerate all subsets of $S$ so that the time complexity is $O(3^n)$, where $n$ is the number of tables:

```cpp
// T is the subset of S.
for (int T = (S - 1) & S; T != 0; T = (T - 1) & S) {
  // ...
}
```

You need to implement it in the `CostBasedOptimizer::Optimize` function in `plan/cost_based_optimizer.hpp`. The DP algorithm is executed if and only if the condition is satisfied (the number of tables is smaller than 20) and the option `enable_cost_based` is set to true. If it is not satisfied, then we simply apply `ConvertToHashJoinRule` on the naive plan.

For two table sets $S$ and $T$, you need to check if they can use hash join. You need to collect all the predicates in the plan tree and check if there is a predicate that can be used for hash keys. More specifically, first, you need to traverse the plan tree and collect the `PredicateVec` objects in join plan nodes. (Since filter plan node has been pushed down by `PushDownFilterRule` (refer to `plan/rules/push_down_filter.hpp`) in LogicalOptimizer::Optimize (refer to `plan/logical_optimizer.cpp`), you do not need to consider filter plan node.) 

Each element in `PredicateVec` is a binary condition expression, refer to `plan/plan_expr.hpp`. For each predicate element in `PredicateVec`, you can check if it can be used for hash keys by the table bitsets. The table bitset is a binary number representing the table set, in which the $i$-th bit is 1 if and only if the $i$-th table is in the table set. Suppose the table bitset of $S$ is $bs$ and the table bitset of $T$ is $bt$, you can check as follows:

```cpp
PredicateElement a;
L = bs;
R = bt;
// only equal condition can use it
if (a.expr_->op_ == OpType::EQ) {
  if (!a.CheckRight(L) && !a.CheckLeft(R) && a.CheckRight(R) &&
      a.CheckLeft(L)) {
    return true;
  }
  if (!a.CheckLeft(L) && !a.CheckRight(R) && a.CheckRight(L) &&
      a.CheckLeft(R)) {
    return true;
  }
}
```

To get the table bitset of $S$, you can enumerate all the tables in $S$ and bitwise-OR the `table_bitset_` field in their sequential scan nodes. In sequential scan nodes, the `table_bitset_` field is a one-hot vector representing the table itself. You can also enumerate table sets from small to large, and store the result of subsets, so that you can just perform one bitwise-OR operation to calculate the new table set using old table set results.

After executing the DP algorithm, you need to create a new plan. You need to create plan nodes based on your DP result.

You can create a nested loop join plan node as follows:

```cpp
auto join_plan = std::make_unique<JoinPlanNode>();
join_plan->table_bitset_ = /* the table bitset of the tables in the subtree */
join_plan->ch_ = /* build table (left) */
join_plan->ch2_ = /* probe table (right) */
join_plan->output_schema_ = OutputSchema::Concat(
        join_plan->ch_->output_schema_, join_plan->ch2_->output_schema_);
join_plan->predicate_ = /* predicate, can be empty */
```

You can create a hash join plan node as follows: (About how to generate left/right hash expressions, you can refer to `plan/rules/convert_to_hash_join.hpp`).

```cpp
auto hashjoin_plan = std::make_unique<HashJoinPlanNode>();
hashjoin_plan->table_bitset_ = /* the table bitset of the tables in the subtree */
hashjoin_plan->ch_ = /* build table (left) */
hashjoin_plan->ch2_ = /* probe table (right) */
hashjoin_plan->output_schema_ = OutputSchema::Concat(
        hashjoin_plan->ch_->output_schema_, hashjoin_plan->ch2_->output_schema_);
hashjoin_plan->predicate_ = /* predicate, can be empty */
hashjoin_plan->left_hash_exprs_ = /* hash key of build table (left) */
hashjoin_plan->right_hash_exprs_ = /* hash key of probe table (right) */
```

For the project plan node, you do not need to create a new one because it is the root. You need to point its child to a new join plan node. Do not forget to assign the DP result (i.e. $f(\text{all tables in query})$) to the `cost_` field of the root plan node, it will be used in tests. You do not need to do anything for sequential scan node (predicate has been pushed down).

## Cost function

You can find `scan_cost` and `hash_join_cost` in the `OptimizerOptions`. 

The cost of nested loop join is: $\text{scan cost} \times (\text{build table size}) \times (\text{probe table size})$

The cost of hash join is: $\text{hash join cost} \times ((\text{build table size}) + (\text{probe table size})) + \text{scan cost}\times (\text{output size})$

For example, suppose the output size is 3000, the cost of joining table A (1000 rows), B (2000 rows) is $3000(\text{hash join cost})+3000(\text{scan cost})$ (hash join) or $2000000(\text{scan cost})$ (nested loop join).

We provide the true cardinality for all possible table sets. It is stored in `true_cardinality_hints` in the `OptimizerOptions`. The `true_cardinality_hints` is a `std::optional` variable, you can use `true_cardinality_hints.has_value()` to test if it is valid. If it is valid, it is a `std::vector` contains pairs storing table sets and the true cardinality of the table sets. The table sets are ordered by the number calculated by $\sum_{i\in S} 2^i$ where $S$ is the table set, $i\in S$ means the $i$-th table in the table list is in $S$. For example, for 3 tables `A, B, C`, the content in the vector is: `{(), ({"A"}, ...), ({"B"}, ...), ({"A", "B"}, ...), ({"C"}, ...), ({"A", "C"}, ...), ({"B", "C"}, ...), ({"A", "B", "C"}, ...)}`. You can reorder it if necessary.

## Test

Use `test/test_opm --gtest_filter=EasyOptimizerTest.Join5TablesCrossProduct:EasyOptimizerTest.Join3Tables` to test you code.

These tests are simple. More tests will be added later.
