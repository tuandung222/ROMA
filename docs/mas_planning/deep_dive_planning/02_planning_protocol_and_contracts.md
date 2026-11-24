# 02. Planning Protocol and Contracts

## 2.1 Planner Protocol Surface

ROMA planner protocol is defined by:
- module class: `Planner` (`src/roma_dspy/core/modules/planner.py`)
- signature: `PlannerSignature` (`src/roma_dspy/core/signatures/signatures.py`)
- subtask schema: `SubTask` (`src/roma_dspy/core/signatures/base_models/subtask.py`)

Runtime integration is in:
- `ModuleRuntime.plan_async` and `_create_subtask_graph` (`src/roma_dspy/core/engine/runtime.py`)

## 2.2 Signature-Level Contract

`PlannerSignature`:
- Input:
  - `goal: str`
  - `context: Optional[str]`
- Output:
  - `subtasks: List[SubTask]`
  - `dependencies_graph: Optional[Dict[str, List[str]]]`

`SubTask` fields:
- `goal`: precise subtask objective
- `task_type`: classification in `TaskType`
- `dependencies`: index-string references to prerequisite subtasks
- optional runtime-oriented fields:
  - `result`
  - `context_input`

## 2.3 Runtime Expectation: Index-Based IDs

Planner output dependencies are expected in index space:
- subtask i has logical ID `str(i)`
- dependencies are lists of such IDs

Example expected dependency graph:

```json
{
  "0": [],
  "1": ["0"],
  "2": ["0", "1"]
}
```

The contract intentionally avoids external UUID exposure to the planner.

## 2.4 Why Index Space Exists

Design rationale:
- LLM-friendly stable local identifiers
- deterministic ordering from planner output list
- easy conversion into runtime IDs

Research implication:
- planner is decoupled from runtime internal identity model
- a deterministic binding step becomes mandatory

## 2.5 Binding and Normalization Pipeline

### Step A: Create concrete `TaskNode`s

For each planner subtask, runtime creates a child `TaskNode` with:
- `goal=subtask.goal`
- `task_type=subtask.task_type`
- `parent_id=parent.task_id`
- `depth=parent.depth+1`
- shared `execution_id`

### Step B: Build `index -> task_id` map

Runtime builds mapping before creating dependencies:

```text
index_to_task_id = {
  "0": <uuid-a>,
  "1": <uuid-b>,
  ...
}
```

### Step C: Convert dependency graph to task-id graph

For each key/value pair in planner `dependencies_graph`:
- skip malformed key if not integer index
- skip key if out-of-range
- remove self-dependencies
- skip dependency entries that are malformed/out-of-range
- produce task-id dependencies

### Step D: Create subgraph

`TaskDAG.create_subgraph(parent_id, subtasks, task_id_dependencies)`
- adds children as nodes
- adds validated edges
- stores subgraph in parent DAG
- updates parent with `subgraph_id`

### Step E: Register index mappings in `ContextStore`

Context store persists `subgraph_id, index -> task_id` mapping for later context assembly and dependency resolution.

## 2.6 Robustness Policy on Invalid Planner Outputs

Current policy is non-failing and forgiving:
- malformed dependencies are ignored
- invalid indices are skipped
- cross-DAG dependency references are filtered

Advantages:
- liveness under noisy planner outputs
- fewer catastrophic planning failures

Risks:
- silently weakened dependency graph
- potential semantic mismatch between intended and executed plan

Potential research extension:
- dual-mode planner protocol:
  - strict mode (reject invalid plan)
  - permissive mode (current behavior)

## 2.7 Planning State Transitions

In recursive mode, a typical planning transition sequence:

```text
PENDING -> ATOMIZING -> PLANNING -> PLAN_DONE
```

After `PLAN_DONE`:
- child subgraph executes
- parent transitions to `AGGREGATING`
- parent becomes `COMPLETED` on aggregation success

In event-driven mode, `READY` events route tasks into planning branch based on current status.

## 2.8 Planner Context Protocol

Planner context comes from `ContextManager.build_planner_context`.

It includes:
- fundamental context:
  - overall objective
  - temporal metadata
  - recursion metadata
- planner-specific context:
  - parent results
  - sibling completed results
  - optional artifact references

Notably:
- planner does not receive file-system write paths by default
- this reflects planner's role as decomposition policy, not execution environment actor

## 2.9 Artifact Injection and Planner Information Diet

Planner artifact visibility is controlled by `artifact_injection_mode`:
- `none`
- `dependencies`
- `subtask`
- `full`

This is a high-impact research lever:
- minimal context can improve decomposition abstraction quality
- maximal context can improve local consistency but increase prompt complexity and leakage of irrelevant details

## 2.10 Planner Prompting Layer vs Runtime Contract Layer

ROMA supports planner customization via:
- `signature_instructions`
- `demos`
- custom signatures

Prompt-level guidance can change behavior, but runtime contract layer remains authoritative:
- subtasks/dependency parsing
- DAG creation constraints
- transition semantics

This separation is critical for experimental control in MAS planning research.

## 2.11 Planning Protocol Pseudocode

```python
def planning_protocol(parent_task, dag):
    # 1) call planner model
    plan = planner(goal=parent_task.goal, context=planner_context(parent_task))

    # 2) instantiate child tasks
    child_nodes = [make_tasknode(subtask, parent_task) for subtask in plan.subtasks]

    # 3) normalize dependency graph from index space
    index_to_task = {str(i): child.task_id for i, child in enumerate(child_nodes)}
    dep_graph = normalize(plan.dependencies_graph, index_to_task)

    # 4) materialize nested subgraph
    dag.create_subgraph(parent_task.task_id, child_nodes, dep_graph)

    # 5) bind context index mapping for later dependency context assembly
    register_index_mapping(parent_task.subgraph_id, index_to_task)

    # 6) mark planning done
    parent_task = transition(parent_task, PLAN_DONE)
    return parent_task
```

## 2.12 Protocol Invariants Specific to Planning

### P1. Parent-child ownership invariant

All planner-produced child tasks must have `parent_id == parent.task_id`.

### P2. Subgraph identity invariant

A planning parent has exactly one active `subgraph_id` reference representing its decomposition instance.

### P3. Dependency closure invariant (within subgraph)

All dependency edges in subgraph must resolve to tasks within that subgraph.

### P4. Depth monotonicity invariant

Child task depth is parent depth + 1 (unless force-execution path mutates status policy later).

## 2.13 Contract Gap Observations

The following contract gaps are important for researchers:

1. No explicit hard validation that every `SubTask.dependencies` exactly matches `dependencies_graph`.
   - Runtime uses `dependencies_graph` for DAG edge creation.

2. No explicit planner output quality score before execution fan-out.
   - The system executes unless structural conversion fails catastrophically.

3. Enum-level support for `NEEDS_REPLAN` exists but replan protocol is not the dominant path in current orchestration implementation.

## 2.14 Suggested Protocol Hardening Experiments

### Experiment A: strict planner schema gate

Add explicit gate before `_create_subtask_graph`:
- reject missing/invalid `task_type`
- enforce dependency index domain
- enforce acyclicity in index graph pre-conversion

Measure:
- failure rate
- downstream execution quality
- average retries

### Experiment B: planner graph quality heuristic

Score plan on:
- width/depth ratio
- number of disconnected components
- critical path length
- orphan task count

Use score to:
- trigger replan
- switch planning prompt variant

### Experiment C: dependency repair policy

Try automatic repair before rejection:
- drop invalid deps
- infer likely dependencies from textual references
- force terminal synthesis node when missing

## 2.15 Minimal Formal Contract for External Planner Interop

For integrating alternative planners with ROMA runtime, a minimal interoperable contract is:

```text
Input:
  goal: string
  context: optional string

Output:
  subtasks: list of objects with fields {goal, task_type, dependencies?}
  dependencies_graph: optional map index-string -> list[index-string]

Constraints:
  index domain = [0, len(subtasks)-1]
  graph must be acyclic or repairable
```

This contract enables experimentation with non-LLM planners while preserving ROMA runtime semantics.

