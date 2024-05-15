# Part 1

In part 1, you will implement `JoinVecExecutor` and `HashJoinVecExecutor`. 

To start, add two new files `execution/vec/hashjoin_vexecutor.hpp` and `execution/vec/join_vexecutor.hpp`. Make sure to include them in `execution/executor.cpp`. 

## JoinPlanNode and HashJoinPlanNode

In `JoinPlanNode`, the `predicate_` member stores join predicates decomposed by `AND` (e.g., `select * from A, B where a.id = b.id AND a.name = b.name AND (a.age < b.age or a.cost > b.cost);` -> an array of size 3 `{a.id = b.id, a.name = b.name, a.age < b.age or a.cost > b.cost}`), you can use `PredicateVec::GenExpr` to generate a `std::unique_ptr<Expr>`. The `output_schema_` member (which is inherited from its parent class) stores the output schema, `ch_` and `ch2_` stores pointers to its two children. Its `type_` is `PlanType::Join`. 

In `HashJoinPlanNode`, the `predicate_`, `output_schema_`, `ch_`, `ch2_` members are as the same as `JoinPlanNode`. However, it has two arrays of expressions that store the hash keys. Hash keys are expressions like `A = B` where `A` only contains attributes from one side and `B` only contains attributes from the other side. These keys are used to build a hash table for the join operation. For example, in `select * from A, B where A.id = B.id`, `A.id` and `B.id` are hash keys. `ch_` is considered as the left table (build table), and `ch2_` is considered as the right table (probe table). The `left_hash_exprs` array stores `A.id` and `right_hash_exprs` array stores `B.id`. Its `type_` equals to `PlanType::HashJoin`. 

During the optimization stage, a `JoinPlanNode` can be converted to a `HashJoinPlanNode` when it has hash keys.

## In execution/executor.cpp

You need to add code to generate `JoinVecExecutor` for `JoinPlanNode` and `HashJoinVecExecutor` for `HashJoinPlanNode` in `InternalGenerateVec`. Ensure that you pass necessary parameters from plan nodes to executors. For example, in `ProjectPlanNode`, `project_plan->output_exprs_` and `project_plan->ch_->output_schema_` are passed to `ProjectVecExecutor`. 

Since join executors have 2 child executors, you will need to call `InternalGenerateVec` recursively to get the executor for `ch_` and `ch2_`. 

## Implementing HashJoinVecExecutor and JoinVecExecutor

Create two classes that inherit from `VecExecutor`. In the constructors, pass `ExecOptions` to `VecExecutor`, which you can get using `db.GetOptions().exec_options`. Then implement `VecExecutor::InternalNext`, which should return a `TupleBatch` each time it is called. It should return an empty result if and only if there is no result to return. In addition, the size of the returned `TupleBatch` cannot exceed the maximum batch size (refer to `ExecOptions::max_batch_size` in `execution/execoptions.hpp`, it is 1024 by default). 

For `JoinVecExecutor`, you need to implement nest loop join. For `HashJoinVecExecutor`, you need to implement hash join. You can review them in the lectures.

For `JoinVecExecutor`, you need to store all the tuples from the build table. You can just use an array of `TupleBatch` to store them, or you can utilize the dynamic size mechanism of `TupleBatch`. Each time you fetch a `TupleBatch` from the probe table and you can just calculate the join predicate for each tuple from the probe table and a batch of tuples from the build table. You do not need to worry about the efficiency when the build table size is small compared to the probe table size. For example, if the build table has only 1 tuple but the probe table has 1000 tuples, the algorithm above will evaluate the predicate for 1 tuple from the build table and 1 tuple from the probe table 1000 times. We assume that the build table is large.

For `HashJoinVecExecutor`, you need to read all the tuples from the build table, and build a hash table. You can just use `std::unordered_map` for this purpose. For the hash function, you can use `utils::Hash` and `utils::Hash8`. `utils::Hash` can hash any data and `utils::Hash8` can only hash a 8-byte integer. For float numbers, you can just use `utils::Hash8` to hash the binary representation. Specifically, `TupleBatch::Get` returns a `StaticFieldRef`, you can use `StaticFieldRef::ReadInt` to read a 8-byte integer from the object, even if its type is float. For strings, you need to use `StaticFieldRef::ReadStringView()` to read the string view, and use `utils::Hash`. To hash more than 1 elements, you can pass the hash value as the seed parameter, for example:

```c++
seed = 0x1234;
hash = utils::Hash8(114514, seed);
hash = utils::Hash8(1919810, hash);
...
```

## Test

You can use `test/test_exec --gtest_filter=ExecutorJoinTest.*` to test your code.
