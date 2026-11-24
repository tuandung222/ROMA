# 04. Failure, Recovery, and Checkpoint Protocol

## 4.1 Why Recovery Is a First-Class Planning Concern

In hierarchical MAS planning, failure is not a binary outcome.
A parent plan can contain:
- completed children with valid outputs
- failed children requiring retries/replanning

ROMA encodes this explicitly with checkpoint snapshots and recovery plans.

Primary implementation:
- `src/roma_dspy/resilience/checkpoint_manager.py`
- `src/roma_dspy/types/checkpoint_models.py`
- `src/roma_dspy/types/checkpoint_types.py`

## 4.2 Checkpoint Data Model

`CheckpointData` includes:
- identity and trigger metadata
- full DAG snapshot (`root_dag`)
- failure set (`failed_task_ids`)
- preserved results
- solver/module state bundles
- optional tool invocation history

DAG snapshot is recursive (`DAGSnapshot` with nested `subgraphs`), enabling restoration of hierarchical decomposition state.

## 4.3 Trigger Semantics

Checkpoint triggers (`CheckpointTrigger`):
- `EXECUTION_START`
- `EXECUTION_COMPLETE`
- `BEFORE_PLANNING`
- `AFTER_PLANNING`
- `BEFORE_AGGREGATION`
- `ON_FAILURE`
- `PERIODIC`
- `MANUAL`

### Trigger interpretation

- `BEFORE_PLANNING`: captures pre-decomposition state before expensive expansion
- `AFTER_PLANNING`: captures concrete decomposition graph
- `BEFORE_AGGREGATION`: captures complete child outputs before synthesis
- `ON_FAILURE`: captures failure frontier for targeted retry

These trigger placements are highly aligned with hierarchical planning failure modes.

## 4.4 Storage Strategy

Checkpoint manager uses file storage as primary.
Optional Postgres dual-write exists when storage is configured.

Properties:
- atomic write via temp file + rename
- optional gzip compression
- integrity verification on load
- expiration cleanup by age and count constraints

## 4.5 Recovery Strategy Types

`RecoveryStrategy`:
- `PARTIAL`: retry failed tasks and affected dependents
- `FULL`: restart all tasks
- `SELECTIVE`: user-defined scope

### Partial strategy logic

For partial recovery, affected set is computed by:
- initializing set with failed tasks
- adding tasks that depend on already affected tasks

This approximates dependency-closure-based invalidation.

## 4.6 Recovery Plan Structure

`RecoveryPlan` includes:
- `tasks_to_retry`
- `tasks_to_preserve`
- toggles:
  - restore DAG state
  - restore module states
  - reset retry counters
  - apply backoff

This structure is sufficient for policy experiments around partial recomputation.

## 4.7 Apply-Recovery Semantics

`CheckpointManager.apply_recovery_plan(...)` performs:
1. load checkpoint
2. optionally restore preserved results/statuses
3. optionally reset retries
4. prepare selected tasks for retry (`READY`-like re-entry)
5. optionally restore module/scheduler state bundles

In event-driven mode, controller can apply pending restoration state for scheduler/event loop metrics and queue-related runtime state.

## 4.8 Interaction with Orchestration Modes

### Recursive mode

- checkpoints created around key transitions
- recovery typically invoked manually or through solver helper methods

### Event-driven mode

- additional event-level checkpointing cadence
- controller supports pending restoration application

Research implication:
- event-driven mode offers a richer substrate for studying stateful recovery under concurrent scheduling.

## 4.9 Recovery Correctness Considerations

### C1. State restoration validity

Restoration may bypass normal transition semantics via `TaskNode.restore_state`.
This is intentional for checkpoint replay but requires auditability.

Mitigation in implementation:
- restoration inserts transition metadata (`restored`, `checkpoint_recovery`)

### C2. Retry counter semantics

Policy can optionally reset retries; this changes effective retry budget post-recovery.

### C3. Preserved result serializability

Only serializable results are preserved in checkpoint metadata.
Non-serializable outputs are skipped.

## 4.10 Checkpoint Protocol Pseudocode

```python
def create_checkpoint(trigger, dag, failure_set=None):
    snap = serialize_dag_recursive(dag)
    preserved = collect_serializable_completed_results(dag)
    data = {
        "trigger": trigger,
        "root_dag": snap,
        "failed_task_ids": failure_set,
        "preserved_results": preserved,
    }
    atomic_write(data)
```

```python
def recover(checkpoint_id, strategy):
    cp = load(checkpoint_id)
    plan = make_recovery_plan(cp, strategy)
    restore_preserved(plan.tasks_to_preserve)
    prepare_retry(plan.tasks_to_retry)
    return restored_dag
```

## 4.11 Observed Design Trade-offs

### Trade-off A: restore fidelity vs operational simplicity

- Full restoration fidelity requires module/scheduler state replay
- Simpler restoration can recompute scheduler state from DAG readiness

ROMA leans toward recomputation where feasible (e.g., queue events regenerated from DAG state).

### Trade-off B: strict consistency vs high availability

- checkpoint failures are often logged with non-fatal behavior to keep execution moving
- this favors availability but can reduce forensic completeness

## 4.12 Recovery-Focused Research Experiments

### Experiment 1: recovery cost-benefit frontier

Compare `PARTIAL` vs `FULL` under varying failure rates.

Metrics:
- time-to-recovery
- incremental token cost
- output quality degradation
- repeated-failure probability

### Experiment 2: trigger placement optimization

Ablate trigger sets:
- sparse (`ON_FAILURE` only)
- strategic (`BEFORE/AFTER_PLANNING`, `BEFORE_AGGREGATION`, `ON_FAILURE`)
- dense (+ periodic)

Measure:
- storage overhead
- recovery success
- replay confidence

### Experiment 3: dependency-closure precision

Replace current affected-task inference with graph-theoretic transitive closure and compare outcomes.

### Experiment 4: deterministic replay quality

Evaluate whether checkpoint + trace data is sufficient to reproduce decision trajectories.

## 4.13 Practical Recommendations

1. Keep `AFTER_PLANNING` enabled in planning-heavy workloads.
2. Keep `ON_FAILURE` enabled for non-trivial fan-out DAGs.
3. Use partial recovery as default for cost-sensitive settings.
4. Add quality gates before preserving outputs if your tasks return large structured artifacts.

