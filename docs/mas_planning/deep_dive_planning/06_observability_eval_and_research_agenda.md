# 06. Observability, Evaluation, and Research Agenda

## 6.1 Why Observability Is Central to Planning Research

For MAS planning/orchestration systems, outputs alone are insufficient.
You need to observe:
- decomposition quality
- orchestration decisions
- failure/retry trajectories
- resource dynamics (tokens, latency, queue pressure)

ROMA exposes multiple telemetry surfaces suitable for rigorous evaluation.

## 6.2 Telemetry Surfaces in ROMA

### Surface A: module execution history on each task

`TaskNode.execution_history` stores `ModuleResult` entries per module:
- input/output
- duration
- token metrics
- metadata

### Surface B: LM traces

Persisted LM traces include:
- module name (planner, executor, ...)
- prompt/response
- token usage
- latency
- model, temperature, cost

API entry points:
- `/api/v1/executions/{execution_id}/lm-traces`
- `/api/v1/executions/{execution_id}/lm-traces/cost-summary`

### Surface C: checkpoint snapshots

Checkpoints provide time-sliced DAG state and recovery metadata.

API entry points:
- `/api/v1/executions/{execution_id}/checkpoints`
- `/api/v1/checkpoints/{checkpoint_id}`
- `/api/v1/executions/{execution_id}/checkpoint` (latest)

### Surface D: execution data consolidation

`/api/v1/executions/{execution_id}/data` consolidates trace-oriented execution view via `ExecutionDataService`.

### Surface E: toolkit and execution event traces

If enabled, toolkit lifecycle and invocation traces are persisted and queryable from metrics endpoints.

## 6.3 Event Emission Semantics: Declared vs Observed

`ExecutionEventType` enumerates rich event vocabulary, but observed emission in `RecursiveSolver` currently includes mostly:
- `execution_start`
- `execution_complete`
- `execution_failed`

Research caution:
- do not assume all enum event types are currently emitted in production paths.
- verify actual emission sites before designing event-driven evaluation protocols.

## 6.4 Evaluation Taxonomy for Planning/Orchestration

We propose a three-axis taxonomy:

1. **Plan quality axis**
   - decomposition minimality
   - dependency correctness
   - redundancy and coverage

2. **Orchestration efficiency axis**
   - makespan
   - queue utilization/overflow
   - retry overhead

3. **Reliability axis**
   - failure containment
   - recovery success
   - replay/audit fidelity

## 6.5 Core Metrics (Implementation-Compatible)

### Planning metrics

- `subtask_count`
- `dependency_edge_count`
- invalid dependency drop rate (can be instrumented around normalization)
- plan depth increment profile

### Execution metrics

- total module calls by type
- total and per-module latency
- total and per-module token/cost

### Orchestration metrics

- queue overflow count (event-driven)
- dropped-event counts by type
- retries per task type
- completion ratio per depth band

### Recovery metrics

- checkpoints per execution
- recovery plan size (`tasks_to_retry`, `tasks_to_preserve`)
- successful recovery rate
- recovery re-failure rate

## 6.6 Derivable Composite Indicators

### Indicator I: Decomposition Efficiency Ratio (DER)

```text
DER = useful_subtasks / total_subtasks
```

Approximate "useful" as subtasks that produce non-empty results and contribute to aggregated output.

### Indicator II: Orchestration Overhead Factor (OOF)

```text
OOF = orchestration_time / total_execution_time
```

Can be approximated from event timing + module duration sums.

### Indicator III: Recovery Leverage Score (RLS)

```text
RLS = (cost_full_restart - cost_partial_recovery) / cost_full_restart
```

## 6.7 Minimal Reproducible Evaluation Workflow

1. Execute task set with fixed config profile and seed prompts.
2. Export traces/checkpoints.
3. Compute plan graph metrics from checkpoint DAG snapshots.
4. Compute execution/resource metrics from LM traces.
5. For event-driven studies, add queue metrics and dropped-event analysis.

CLI helpers:

```bash
roma-dspy solve "<task>" --profile general
roma-dspy exec export <execution_id> --level full
roma-dspy checkpoint list <execution_id>
```

API helpers:

```bash
curl "http://localhost:8000/api/v1/executions/<id>"
curl "http://localhost:8000/api/v1/executions/<id>/checkpoint"
curl "http://localhost:8000/api/v1/executions/<id>/lm-traces?module_name=planner"
```

## 6.8 Research Agenda: Planning/Policy/Protocol

### Agenda A: protocol strictness studies

Compare permissive vs strict planner-output validation.

Question:
- does strict rejection improve final quality enough to justify liveness loss?

### Agenda B: adaptive decomposition policy

Introduce runtime feedback to planner prompt:
- queue pressure
- retry/failure signals
- remaining depth budget

Question:
- can planner self-regularize fan-out under orchestration stress?

### Agenda C: replan policy integration

`NEEDS_REPLAN` status exists in type model but can be more deeply integrated.

Question:
- what replan trigger/policy yields best cost-quality trade-off?

### Agenda D: planner-verifier coupling

Add verifier checkpoint between planning and execution fan-out.

Question:
- can plan verification reduce downstream retries/failures?

### Agenda E: recovery-aware planning

Inject checkpoint/recovery constraints into planner objective.

Question:
- can planner produce failure-containment-friendly decomposition structures?

## 6.9 Experimental Controls and Confounders

Control variables:
- model/provider and temperature per agent
- planner prompt and demos
- max depth and max concurrency
- artifact injection mode

Confounders:
- external API/tool variability
- provider latency spikes
- optional persistence/observability toggles

Recommendation:
- isolate planning experiments with deterministic/local tools where possible.

## 6.10 Suggested Benchmark Task Classes

1. shallow atomic tasks (control)
2. medium decomposition tasks with moderate dependencies
3. high fan-out tasks stressing scheduler and queue
4. failure-injection tasks targeting recovery pathways
5. long-horizon tasks with deep recursion pressure

## 6.11 High-Value Instrumentation Additions

1. planner normalization diagnostics:
   - count invalid keys/deps dropped
   - emit structured repair report in module metadata

2. explicit planner graph metrics at plan time:
   - width, depth estimate, connected components, critical path estimate

3. event-level causal trace IDs:
   - connect emitted events to originating module call and checkpoint IDs

4. canonical replay bundle:
   - checkpoint + event sequence + module history for deterministic replay research

## 6.12 Practical Publication-Grade Study Design

A publication-grade study could be structured as:

- baseline: current ROMA planner + recursive orchestration
- treatment 1: strict planner protocol gate
- treatment 2: adaptive fan-out planner prompt
- treatment 3: event-driven scheduler + priority variants
- treatment 4: recovery-aware planner policy

Primary outcomes:
- quality (task success, hallucination/error proxy, verifier pass rate)
- efficiency (latency, token/cost)
- robustness (failure recovery and rerun stability)

## 6.13 Closing Observation

ROMA already exposes enough structural hooks to support serious MAS planning/orchestration research without redesigning the core runtime.

The strongest near-term gains are likely from:
- planner protocol hardening
- adaptive decomposition policies tied to scheduler telemetry
- richer recovery-aware planning objectives

