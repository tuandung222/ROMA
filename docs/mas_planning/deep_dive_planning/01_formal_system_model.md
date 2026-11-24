# 01. Formal System Model

## 1.1 Motivation

In MAS planning research, ambiguity usually appears at three levels:
- task state semantics
- graph/dependency semantics
- controller policy semantics

ROMA's implementation allows a reasonably formal model that maps directly to code-level objects.

## 1.2 Core Objects as Mathematical Entities

Let an execution instance be:

```text
E = (T_root, G_root, S, C, O)
```

where:
- `T_root`: root task node (`TaskNode`)
- `G_root`: root DAG (`TaskDAG`)
- `S`: solver/controller policy (`RecursiveSolver` or event loop controller)
- `C`: context system (`ContextManager`, `ContextStore`, execution context)
- `O`: observability state (LM traces, events, checkpoints)

### Task node model

Each task node is a tuple:

```text
T = (id, parent, goal, task_type, node_type, status, depth, deps, children, result, metrics, hist)
```

Code correspondence:
- `TaskNode` in `src/roma_dspy/core/signatures/base_models/task_node.py`
- immutable updates through `model_copy()` and transition helpers

### DAG model

A DAG instance:

```text
G = (V, E_dep, subgraphs, metadata)
```

where:
- `V`: task IDs mapped to immutable `TaskNode`s
- `E_dep`: directed dependency edges (pred -> succ)
- `subgraphs`: map `subgraph_id -> TaskDAG`
- `metadata`: creation/execution metadata

## 1.3 State Space

Define task status domain:

```text
Sigma = {
  PENDING,
  ATOMIZING,
  PLANNING,
  PLAN_DONE,
  READY,
  EXECUTING,
  AGGREGATING,
  COMPLETED,
  FAILED,
  NEEDS_REPLAN
}
```

Terminal states under current semantics:

```text
Terminal = {COMPLETED, FAILED}
```

`TaskDAG.is_dag_complete()` evaluates true iff every task in root and nested subgraphs is terminal.

## 1.4 Transition Relation

A task transition is valid only if allowed by `TaskStatus.can_transition_to`.

Formal relation:

```text
delta: Sigma x Event -> Sigma
```

with constraints encoded in `TaskStatus` and enforced by `TaskNode.transition_to`.

Important implementation property:
- transitions are validated and audited (`StateTransition` list)
- invalid transitions throw, preventing silent policy drift

## 1.5 Decomposition Morphism

Planning defines a decomposition morphism:

```text
D: T_parent -> G_sub
```

where:
- input task has `status=PLANNING`
- planner outputs `n` subtasks + dependency map
- runtime builds `n` concrete child `TaskNode`s and a nested DAG
- parent gets `subgraph_id` and transitions to `PLAN_DONE`

This morphism is implemented by:
- `ModuleRuntime.plan_async` + `ModuleRuntime._create_subtask_graph`
- `TaskDAG.create_subgraph`

## 1.6 Execution Readiness Predicate

Task `t` in DAG `G` is ready iff:

```text
ready(t) := status(t) in {PENDING, READY} and all predecessors of t are COMPLETED
```

Code correspondence:
- `TaskDAG._dependencies_satisfied`
- `TaskDAG.iter_ready_nodes`

Note: planning and aggregation are handled by status-specific controller logic, not simple dependency-only readiness.

## 1.7 Hierarchical Completion Condition

For child task `c` with parent `p`:
- when `c` completes, controller checks whether parent subgraph is globally complete
- if complete, parent is eligible for aggregation

Code correspondence:
- `TaskDAG.check_subgraph_complete`
- event loop emits `SUBGRAPH_COMPLETE`

## 1.8 Invariants

### Invariant I1: DAG acyclicity

All dependency additions preserve DAG acyclicity.

Enforced by:
- `TaskDAG.add_edge` cycle check with rollback
- `TaskDAG._validate_dag_integrity`

### Invariant I2: Task identity stability

Task identity must remain stable across immutable copies.

Enforced by:
- `TaskNode.model_copy` override preserving `task_id`

This is critical for planner materialization correctness.

### Invariant I3: Transition safety

No task may jump to an illegal status.

Enforced by:
- `TaskNode.transition_to` + `TaskStatus.can_transition_to_status`

### Invariant I4: Dependency references in subgraph

Dependencies inside a subgraph must refer to nodes in that subgraph; cross-DAG deps are filtered.

Enforced by:
- `TaskDAG.create_subgraph` filtering logic

### Invariant I5: Execution isolation

Each new root DAG gets its own execution context and file storage scope.

Enforced by:
- `RecursiveSolver._initialize_task_and_dag`
- `ExecutionContext.set(execution_id, file_storage)`

## 1.9 Policy-Level Semantics

Two controller semantics share the same task/DAG representation:

1. Recursive semantics (`_async_execute_state_machine`)
2. Event-queue semantics (`EventLoopController` + `EventScheduler`)

Both must preserve invariants I1-I5 while differing in scheduling and concurrency behavior.

## 1.10 Concurrency Model

### Recursive mode

- subgraph children can run in parallel within `ModuleRuntime.solve_subgraph_async`
- readiness computed from dependency closure
- uses `asyncio.gather` for ready set

### Event-driven mode

- global event queue with configurable worker concurrency
- priority defaults to task depth (`priority_fn` optional)
- explicit dropped-event handling under queue overflow

## 1.11 Cost Model Skeleton

A useful abstract cost decomposition:

```text
Cost(E) = C_atomize + C_plan + C_execute + C_aggregate + C_retry + C_recovery + C_overhead
```

Where each component is measurable from traces/checkpoints:
- module durations (`NodeMetrics` / `ModuleResult.duration`)
- token/cost metrics (`TokenMetrics`, LM traces)
- retry counts and delay policy
- checkpoint persistence overhead

## 1.12 Formal View of Planner Interface

Planner contract can be represented as:

```text
P(goal, context) -> (subtasks, dep_graph)
```

with post-conditions:
- subtasks indexed `0..n-1`
- dependency references resolve to this index domain
- effective runtime dependency graph is acyclic after normalization/filtering

Normalization function:

```text
N(index_graph, index_to_task_id) -> task_id_graph
```

N is partial on malformed indices but total in implementation via skip-on-invalid behavior.

## 1.13 Correctness vs Robustness Distinction

ROMA chooses robust execution over strict planner rejection:
- malformed dependency indices are ignored rather than fatal
- invalid edges filtered out

Implication:
- the runtime is resilient to noisy planner outputs
- but may execute a weakened dependency structure if planner output quality degrades

For MAS research, this is a key policy trade-off between strict protocol adherence and liveness.

## 1.14 Minimal Operational Semantics (Pseudo-formal)

```text
Given task t in DAG G:

1) if depth(t) >= max_depth(t): execute(t)
2) else atomize(t)
3) if node_type(t)=PLAN: plan(t) -> subgraph(t)
4) run ready children in subgraph respecting dependencies
5) when subgraph complete: aggregate(parent)
6) mark terminal or retry/fail per failure policy
```

This semantics is preserved in both orchestration variants with different control mechanisms.

## 1.15 Research Hooks Exposed by the Model

The formal structure exposes clear intervention points:

- Replace `delta` policy (state transition policy extensions)
- Replace readiness priority function
- Add strict planner schema validator before normalization
- Add plan repair pass before subgraph materialization
- Add formal checks for dependency completeness and critical-path quality

