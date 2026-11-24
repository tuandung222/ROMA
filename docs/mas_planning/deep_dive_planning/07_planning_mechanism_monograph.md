# 07. Planning Mechanism Monograph (Super Deep-Dive)

## 7.1 Thesis

ROMA's planning mechanism can be interpreted as a **typed decomposition protocol** with:
- LLM-facing symbolic index graph production
- runtime-facing stable identity graph materialization
- hierarchical execution semantics over nested DAGs

At its core, planning is not "generate subtasks"; it is:

```text
(Goal, Context, Policy) -> (Executable Subgraph, Parent Aggregation Contract)
```

where the output must survive normalization, transition constraints, scheduler policies, and failure/recovery pathways.

## 7.2 End-to-End Planning Pipeline

### Phase 0: precondition setup

Planning is reachable only after:
1. root/child task is initialized with valid execution context
2. atomizer sets node type to `PLAN`
3. task state reaches `PLANNING`

Key gate conditions:
- `task.depth < task.max_depth` (else forced execute)
- valid transition path into planning state

### Phase 1: planner invocation contract

Planner call in runtime (`plan_async`) is effectively:

```python
result = planner.aforward(goal=task.goal, context=planner_context_xml)
```

with traced execution wrapper providing:
- span metadata
- token extraction
- module result recording

### Phase 2: planner output capture

Planner output is recorded in task execution history as module result payload:
- full `subtasks` dump
- raw `dependencies_graph`

This guarantees post-hoc inspectability of planner raw output even after normalization.

### Phase 3: subtask graph construction

`_create_subtask_graph` performs runtime binding:

1. instantiate child `TaskNode`s
2. create `index -> task_id` map
3. convert index dependency graph into task-id dependency graph
4. create nested DAG via `TaskDAG.create_subgraph`
5. register index mapping in `ContextStore`
6. update parent metrics and `subgraph_id`

### Phase 4: planning completion signal

Parent task transitions to `PLAN_DONE`.
This is the synchronization boundary:
- decomposition completed
- execution fan-out now admissible

### Phase 5: downstream execution and aggregation

Children are executed according to readiness policy.
Parent aggregates only after subgraph completion condition is met.

## 7.3 Typed Decomposition Semantics

Each planner-produced subtask carries `task_type`.
This is more than taxonomy; it affects policy downstream via:
- retry/backoff policy selection
- possibly task-aware agent mapping (if configured)

Hence planning outputs are not purely structural; they are typed control hints to runtime policy.

## 7.4 Dependency Semantics in Two Spaces

### Symbolic space (planner-facing)

Graph `G_i = (V_i, E_i)` where:
- `V_i = {"0", ..., "n-1"}`
- edges over index strings

### Runtime space (engine-facing)

Graph `G_r = (V_r, E_r)` where:
- `V_r = {uuid_0, ..., uuid_n-1}`
- edges over concrete `task_id`

Binding morphism:

```text
B: V_i -> V_r
B("k") = task_id_of_subtask_k
```

Edge mapping:

```text
(u, v) in E_i  =>  (B(u), B(v)) in E_r
```

only if both endpoints survive validation.

## 7.5 Validation and Repair Semantics

Current implementation applies local repair by omission:
- drop invalid subtask keys
- drop invalid dependency indices
- drop self-dependencies
- drop out-of-range references
- filter cross-subgraph dependencies

This creates a repaired graph `G_r'`.

Important research implication:
- `G_r'` may be a strict subgraph of planner-intended graph `G_i`.

## 7.6 Parent-Child Contract and Aggregation Boundary

Parent planning node maintains:
- `subgraph_id`
- execution history including planner payload
- eventual aggregation contract over child results

Aggregation boundary condition:
- only when child subgraph is terminal (`is_dag_complete` true)

Hence planning commits the parent to a future aggregation step and defines its input domain.

## 7.7 Planning Context as Control Signal

Planner context currently includes:
- global objective alignment
- recursion budget (`current_depth`, `max_depth`, `at_limit`)
- parent/sibling historical results
- optional artifacts by policy mode

This means planner is not stateless.
It is conditioned on hierarchical execution history and policy-level context visibility.

## 7.8 Planning and Recursion Budget

Depth budget enforces decomposition horizon.
At depth limit, planner is bypassed and task is forcibly executed.

This creates an implicit policy:
- decomposition value must be realized before boundary depth
- late-stage tasks collapse into execution even if planner would decompose

Potential side effect:
- complex subgoals near depth boundary may be under-decomposed.

## 7.9 Planning in Recursive vs Event-Driven Controllers

### Recursive controller

Planning appears as direct branch in state machine.
After planning, children execute and parent aggregates in structured recursive flow.

### Event-driven controller

Planning occurs in `READY` handler when task status is `PLANNING`.
After planning:
- if children exist: enqueue ready tasks
- else emit subgraph completion event

This explicit event semantics makes planning completion an eventful transition rather than implicit call stack continuation.

## 7.10 Planning Failure Surfaces

Possible failure loci:
1. planner module call failure (LLM/tool/runtime)
2. malformed planner output severe enough to break construction
3. DAG materialization errors (e.g., cycle edge attempt)
4. downstream failures in child execution due to weak plan

Observed resilience strategy:
- retries and backoff at orchestration layer
- checkpoint capture around planning boundaries

## 7.11 Planning-Aware Recovery Semantics

Planning-specific checkpoints are strategically placed:
- before planning
- after planning

This enables differentiated recovery:
- if failure occurs after decomposition, restore and retry from known plan
- if planning itself is unstable, potentially retry planner from pre-plan snapshot

This is a strong foundation for research on plan stability and robust decomposition.

## 7.12 Formal Invariants for Planning Correctness

### PI1: identity consistency

Child task IDs generated during binding must remain stable through immutable updates.

### PI2: parent ownership consistency

All child nodes in subgraph have `parent_id = parent.task_id`.

### PI3: subgraph edge locality

All dependencies in subgraph reference only subgraph nodes.

### PI4: acyclic execution graph

Subgraph remains DAG after edge insertion and validation.

### PI5: aggregation admissibility

Parent may aggregate only when subgraph is terminal.

## 7.13 Quantifying Plan Structure Quality (Suggested)

Define plan structural vector:

```text
Q_plan = [n_nodes, n_edges, width_max, depth_est, components, invalid_drop_count]
```

Where:
- `invalid_drop_count` should be instrumented during normalization

Potential scores:

```text
S_compactness = n_edges / max(1, n_nodes)
S_parallelism = width_max / max(1, n_nodes)
S_repair = 1 - invalid_drop_count / max(1, attempted_edges)
```

## 7.14 Suggested Planning Policy Extensions

### Extension A: pre-materialization verifier

Introduce `verify_plan(plan)` before DAG creation:
- enforce schema and graph checks
- optionally auto-repair with explicit audit output

### Extension B: replan trigger policy

If repair score below threshold:
- transition task to `NEEDS_REPLAN`
- invoke planner with stricter instructions

### Extension C: critical-path-aware planning

Prompt planner to minimize critical path under dependency constraints.

### Extension D: adaptive fan-out cap

Fan-out cap conditioned on queue pressure and historical retry rate.

## 7.15 Planning Mechanism Pseudocode (Expanded)

```python
def execute_planning_mechanism(parent, dag, runtime):
    # Preconditions
    assert parent.status == PLANNING
    assert not parent.should_force_execute()

    # Build planner context (history-aware, policy-aware)
    ctx = runtime.context_manager.build_planner_context(parent, runtime, dag)

    # Planner inference
    plan = planner.aforward(goal=parent.goal, context=ctx)

    # Record raw planner output for observability
    parent = record_module_result(parent, "planner", output=plan)

    # Instantiate concrete child nodes
    children = []
    for sub in plan.subtasks:
        children.append(make_child_node(parent, sub))

    # Bind symbolic indices -> runtime IDs
    idx2id = {str(i): child.task_id for i, child in enumerate(children)}

    # Normalize dependency graph (repair-by-omission)
    dep_graph = {}
    for idx, deps in (plan.dependencies_graph or {}).items():
        if not valid_index(idx, len(children)):
            continue
        dst = idx2id[idx]
        normalized_deps = []
        for dep in deps:
            if not valid_index(dep, len(children)):
                continue
            if dep == idx:
                continue
            normalized_deps.append(idx2id[dep])
        if normalized_deps:
            dep_graph[dst] = normalized_deps

    # Materialize nested subgraph and enforce DAG constraints
    dag.create_subgraph(parent.task_id, children, dep_graph)

    # Register index mapping for context reconstruction
    register_index_map(parent.subgraph_id, idx2id)

    # Mark planning complete
    parent = parent.transition_to(PLAN_DONE)
    dag.update_node(parent)

    return parent
```

## 7.16 Practical Blueprint for MAS Researchers

If you want to prototype new planning frameworks on top of ROMA:

1. Keep runtime normalization and DAG invariants intact.
2. Experiment at planner protocol layer first (prompt, demos, strictness gate).
3. Add instrumentation for plan repair counts and graph metrics.
4. Compare recursive vs event-driven behavior under same plan outputs.
5. Evaluate recovery-aware planning with strategic checkpoints enabled.

## 7.17 Final Research Takeaway

ROMA's planning mechanism already embodies a clean separation of concerns:
- symbolic planning interface
- deterministic runtime binding
- policy-driven orchestration
- resilience and observability hooks

This makes it a viable platform for serious research in:
- planning protocol design
- orchestration policy optimization
- robust MAS execution under uncertainty

