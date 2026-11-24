# 03. Orchestration Policies and Scheduler Semantics

## 3.1 Two Controllers, One Runtime Substrate

ROMA exposes two orchestration controllers over the same task/DAG/runtime substrate:

1. Recursive controller (`RecursiveSolver.async_solve`)
2. Event-driven controller (`RecursiveSolver.async_event_solve`)

Both rely on:
- `TaskNode`/`TaskDAG` state model
- `ModuleRuntime` module execution primitives

Differences are policy-level:
- scheduling granularity
- concurrency handling
- failure propagation mechanics
- checkpoint trigger timing

## 3.2 Recursive Policy Semantics

Primary execution path in API service currently uses recursive policy.

Policy kernel is `RecursiveSolver._async_execute_state_machine`.

Decision logic:

```text
if force_execute: force executor
else
  if PENDING: atomize
  if ATOMIZING: transition to PLANNING or EXECUTING
  if PLANNING: call planner + create subgraph
  if EXECUTING: call executor
  if PLAN_DONE: solve subgraph children, then aggregate
```

Characteristics:
- compact policy logic
- straightforward reasoning about parent-child lifecycle
- less explicit global queue state

## 3.3 Event-Driven Policy Semantics

Event-driven controller introduces explicit event queue and worker loop.

Core components:
- `EventScheduler` (priority queue + worker pool)
- `EventLoopController` (handlers for READY/COMPLETED/FAILED/SUBGRAPH_COMPLETE)

Lifecycle:
1. enqueue ready tasks from DAG hierarchy
2. workers consume events
3. handlers emit follow-up events
4. completion when DAG terminal and queue drained

Characteristics:
- explicit queueing and event metrics
- configurable concurrency (`max_concurrency`)
- overflow handling and dropped-event policy

## 3.4 Event Types and Handler Policy

`EventType` enum:
- `READY`
- `COMPLETED`
- `FAILED`
- `SUBGRAPH_COMPLETE`
- `STOP`

### READY handler semantics

Given event for task t:
- if force-exec condition holds: force execute -> emit `COMPLETED`
- if `PENDING`: atomize + transition + requeue READY
- if `PLANNING`: plan + enqueue children or emit subgraph complete
- if `EXECUTING`: execute -> emit `COMPLETED`
- if `AGGREGATING`: aggregate -> emit `COMPLETED`

### COMPLETED handler semantics

- check whether parent subgraph became complete
- enqueue newly ready tasks
- emit `SUBGRAPH_COMPLETE` for parent if applicable

### SUBGRAPH_COMPLETE handler semantics

- only acts when parent task is in `PLAN_DONE`
- aggregate parent
- emit `COMPLETED`

### FAILED handler semantics

- checkpoint on failure (if enabled)
- attempt retry with task-type-aware delay
- if retries exhausted: mark failed and propagate completion checks

## 3.5 Priority and Queue Semantics

Default priority function:
- `priority = task.depth`

Lower numeric value is higher queue priority (priority queue min-heap behavior).

Implication:
- shallow tasks tend to run before deep tasks by default
- custom `priority_fn` can impose cost-aware, deadline-aware, or critical-path-aware ordering

## 3.6 Queue Overflow Policy

`EventScheduler` uses bounded queue (`max_queue_size`, default 1000).

On overflow:
- for non-STOP events: drop event, record metrics, notify controller
- for STOP events: attempt forced insertion by dropping low-priority non-STOP event

Controller reaction to dropped events:
- removes corresponding `(dag_id, task_id)` from queued set to allow re-enqueue later

Research significance:
- this is a liveness-preserving mechanism under overload
- but event loss can temporarily perturb fairness and latency

## 3.7 Concurrency Semantics

### Recursive branch-level parallelism

Subgraph solving can parallelize ready child tasks with `asyncio.gather`.

### Event-driven worker-level parallelism

Scheduler spawns `max_concurrency` workers consuming global queue.

Comparative research hypotheses:
- event-driven may provide better throughput under broad DAG fronts
- recursive mode may provide lower control overhead on narrow/deep trees

## 3.8 Failure and Retry Policy

Failure policy integrates:
- resilience decorators (`with_module_resilience`) for module execution wrapper
- task-level retry policy in event loop failed handler

Retry behavior uses task type-specific configs (`RetryPolicy` defaults):
- `RETRIEVE`: more retries with exponential backoff
- `WRITE`/`THINK`: moderate retries
- `IMAGE_GENERATION`: no retry by default

### Permanent failure behavior

If retries exhausted:
- task marked `FAILED`
- parent completion checks continue
- orchestration attempts to progress where possible

## 3.9 Completion Detection

Event scheduler completion condition:
- not stop requested
- DAG complete (`dag.is_dag_complete()`)
- event queue empty

Potential subtlety for research:
- completion depends on interaction between queued set bookkeeping and dropped-event recovery logic.

## 3.10 Policy Pseudocode Comparison

### Recursive policy

```python
def recursive_policy(task):
    task = state_machine_step(task)
    if task.status == PLAN_DONE:
        run_children(task.subgraph)
        task = aggregate(task)
    return task
```

### Event-driven policy

```python
while not done:
    e = dequeue_event()
    e2 = handle(e)
    if e2: enqueue(e2)
```

The event-driven model externalizes controller state as a queue/event stream, making policy instrumentation and intervention easier.

## 3.11 Planning-Orchestration Coupling Points

Critical coupling points between planner quality and orchestration behavior:

1. **Subtask count explosion**
   - affects queue pressure and checkpoint frequency

2. **Dependency sparsity/density**
   - affects ready-set width and parallelism

3. **Invalid dependency output rate**
   - affects graph repair behavior and semantic drift

4. **Depth profile**
   - affects force-execute boundary and priority ordering

## 3.12 Suggested Orchestration Research Experiments

### Experiment 1: policy comparison benchmark

Compare recursive vs event-driven on same task sets.

Metrics:
- makespan
- queue overflow count
- retry rate
- token cost
- plan fidelity (post-hoc)

### Experiment 2: priority policy ablation

Swap default depth priority with:
- critical path estimate
- uncertainty/cost proxy
- fairness-based round-robin buckets by depth

### Experiment 3: backpressure-aware planner

Feed scheduler congestion metrics to planner prompt:
- encourage smaller fan-out under pressure
- evaluate whether dynamic decomposition policy stabilizes queue utilization

### Experiment 4: event-level deterministic replay

Persist full event sequence and implement deterministic replay harness.

Goal:
- isolate policy bugs and regression-test orchestration semantics

## 3.13 Operational Recommendations

1. Use recursive mode for simpler, low-concurrency deployments and baseline studies.
2. Use event-driven mode when studying scheduling policy, queue dynamics, or high-concurrency workloads.
3. Collect queue overflow metrics when fan-out is large; this is an early warning of orchestration stress.
4. Couple planning experiments with orchestration telemetry; planning quality alone is insufficient.

