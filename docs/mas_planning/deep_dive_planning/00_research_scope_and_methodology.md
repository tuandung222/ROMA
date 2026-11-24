# 00. Research Scope and Methodology

## 0.1 Objective

This dossier analyzes ROMA as an implementation of a hierarchical planning/orchestration framework for MAS.

The core research objective is to answer:

1. What is the planning protocol ROMA actually executes at runtime?
2. Which orchestration policies are encoded in the engine, and where?
3. What invariants does the runtime enforce on decomposition and execution?
4. How does the system represent, recover, and observe planning state over time?

The analysis is implementation-grounded and does not infer behavior not supported by code.

## 0.2 Unit of Analysis

The analysis unit is an execution instance identified by `execution_id`, containing:
- one root `TaskNode`
- one root `TaskDAG`
- zero or more nested subgraphs
- module-level traces and checkpoint snapshots

Execution semantics are extracted primarily from:
- `src/roma_dspy/core/engine/solve.py`
- `src/roma_dspy/core/engine/runtime.py`
- `src/roma_dspy/core/engine/dag.py`
- `src/roma_dspy/core/engine/event_loop.py`
- `src/roma_dspy/core/engine/scheduler.py`

## 0.3 Architectural Boundary Conditions

### In-scope

- planning and decomposition behavior
- dependency and subgraph semantics
- orchestration policies in recursive and event-driven modes
- context propagation relevant to planner quality
- retry/recovery/checkpoint behavior
- observability pathways that expose planning behavior

### Out-of-scope

- model-quality claims for specific LLM providers
- benchmark performance claims not reproducible from repository code alone
- business-level product assumptions

## 0.4 Methodological Approach

### A. Static protocol extraction

Protocol extraction is performed by reading:
- signature contracts (`PlannerSignature`, `SubTask`)
- state machine code paths
- DAG materialization and normalization code
- retry and checkpoint code paths

### B. Dynamic semantics reconstruction

Runtime semantics are reconstructed from:
- state transitions in `TaskNode.transition_to`
- orchestration decisions in `RecursiveSolver._async_execute_state_machine`
- event handlers in `EventLoopController._handle_*`

### C. Data model reconstruction

Data contracts are reconstructed from:
- `TaskNode`, `SubTask`, `DAGSnapshot`, `CheckpointData`, `ModuleResult`
- API schemas in `src/roma_dspy/api/schemas.py`

### D. Cross-layer consistency checks

Consistency checks compare:
- planner output assumptions vs normalization logic
- checkpoint representation vs runtime DAG expectations
- event traces enum vs actual event emission sites

## 0.5 Practical Findings That Shape the Rest of This Dossier

1. ROMA supports two orchestration styles:
   - direct recursive state machine (`async_solve`)
   - event-driven queue scheduler (`async_event_solve`)

2. The API execution service currently uses recursive solve by default (`ExecutionService._run_execution` calls `solver.async_solve(...)`).

3. Planning output is index-based (`"0"`, `"1"`, ...) at the signature boundary but translated to stable runtime `task_id` dependencies inside `ModuleRuntime._create_subtask_graph`.

4. Planning nodes are represented by parent tasks with `subgraph_id`, and child tasks live in a separate nested DAG object.

5. Recovery semantics are first-class: checkpoints include DAG snapshots and can generate partial/full/selective recovery plans.

## 0.6 Why This Matters for MAS Research

ROMA is useful as a research substrate because it separates:
- **policy** (state transitions and scheduling)
- **protocol** (planner input/output contracts)
- **representation** (task graph + checkpoints)
- **telemetry** (LM traces, event traces, toolkit traces)

This separation allows controlled intervention studies:
- changing planner prompt/instructions without changing orchestration code
- swapping recursive vs event-driven orchestration with same task contracts
- evaluating recovery policy effects on throughput and quality

## 0.7 Threats to Validity in Code-Centric Analysis

### Threat 1: Prompt drift vs structural guarantees

Planner behavior can drift with model/provider/prompt changes, while structural constraints remain fixed.

Mitigation:
- anchor all claims on structural constraints in runtime normalization and DAG construction.

### Threat 2: Feature flags and optional dependencies

Persistence and observability behavior depends on configuration (Postgres, MLflow, event traces).

Mitigation:
- classify behavior as core engine vs optional instrumentation.

### Threat 3: Enum-level intent vs actual emission

The event type enum lists module-level event names (`plan_complete`, etc.), but not all are emitted in current `RecursiveSolver` implementation.

Mitigation:
- distinguish "declared" vs "observed emission sites" in later chapters.

## 0.8 Reproducible Inspection Checklist

Use this checklist to replicate the analysis:

1. Inspect solver pathways
```bash
rg -n "async_solve|async_event_solve|_async_execute_state_machine" src/roma_dspy/core/engine/solve.py
```

2. Inspect planning materialization
```bash
rg -n "plan_async|_create_subtask_graph|dependencies_graph" src/roma_dspy/core/engine/runtime.py
```

3. Inspect DAG semantics
```bash
rg -n "create_subgraph|iter_ready_nodes|check_subgraph_complete" src/roma_dspy/core/engine/dag.py
```

4. Inspect event-driven policies
```bash
rg -n "_handle_ready|_handle_completed|_handle_failed" src/roma_dspy/core/engine/event_loop.py
```

5. Inspect checkpoint protocol
```bash
rg -n "create_checkpoint|create_recovery_plan|apply_recovery_plan" src/roma_dspy/resilience/checkpoint_manager.py
```

## 0.9 Reading Order Recommendation

If your research focus is:

- planning protocol design: read `02` first, then `01`, then `05`
- orchestration policy design: read `03`, then `04`, then `06`
- reliability and recovery: read `04`, then `03`, then `06`

