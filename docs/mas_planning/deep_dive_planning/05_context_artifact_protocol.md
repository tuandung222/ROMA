# 05. Context and Artifact Protocol for Planning Quality

## 5.1 Context as a Protocol Layer, Not Just Prompt Text

ROMA treats context construction as a first-class protocol:
- data modeling in `core/context/models.py`
- orchestration in `core/context/manager.py`
- execution-scope propagation in `core/context/execution_context.py`

For planning research, this matters because planner behavior is heavily mediated by what context channels are exposed and when.

## 5.2 Context Layering Model

Context payload sent to modules is XML composed from:

1. `fundamental_context`
   - overall objective
   - temporal context
   - recursion context
   - optional file system context (module-dependent)

2. module-specific context
   - planner-specific: parent/sibling results + artifacts
   - executor-specific: dependency results + artifacts
   - aggregator-specific: subtask artifacts

Protocol composition function:

```text
context_xml = <context> + fundamental + module_specific + </context>
```

## 5.3 Planner-Specific Context Semantics

Planner context builder (`build_planner_context`) injects:
- parent result(s), if available
- completed sibling result(s), if available
- artifact references subject to injection mode

Not injected by default for planner:
- file-system operational paths

Interpretation:
- planner is guided by relational task context, not execution-side filesystem mechanics

## 5.4 ContextStore as Dependency Memory Substrate

`ContextStore` stores per-task outputs and mapping metadata:
- `task_id -> result`
- `subgraph_id -> {index -> task_id}`

Uses:
- dependency-context reconstruction for execution and aggregation
- planner/sibling context lookup support
- O(1) result fetch for dependency-aware context assembly

## 5.5 Artifact Injection Modes as Information Policy

Artifact visibility policy (`ArtifactInjectionMode`) is configurable per agent.

Modes:
- `none`: no artifact injection
- `dependencies`: artifacts from dependency tasks
- `subtask`: artifacts scoped to current subtask neighborhood
- `full`: all available artifacts in execution registry

Policy impact on planning:
- low visibility can increase abstraction and reduce noise
- high visibility can improve concrete planning but risks context dilution

## 5.6 ExecutionContext and Metrics Buffers

ExecutionContext stores:
- execution-scoped `FileStorage`
- artifact registry
- buffered events/tool traces for deferred persistence

This enables:
- isolated execution contexts in concurrent environments
- delayed persistence of observability data
- artifact lookup during planning/execution

## 5.7 Information Flow Geometry

The effective information geometry is:

- top-down:
  - overall objective + recursion limits
- lateral:
  - sibling summaries for planner
- bottom-up:
  - dependency results for executor
- cross-cutting:
  - artifact references via injection policy

Planning quality depends on balancing these channels.

## 5.8 Context Protocol Pseudocode

```python
def build_planner_context(task, runtime, dag, mode):
    fundamental = build_fundamental(task, include_file_system=False)

    parent_results = lookup_parent_result(task.parent_id)
    sibling_results = lookup_completed_siblings(task.parent_id)

    artifacts = query_artifacts(
        task_ids=[task.parent_id] if task.parent_id else [],
        mode=mode,
        current_task_id=task.task_id,
        dag=dag,
    )

    planner_specific = {
        "parent_results": parent_results,
        "sibling_results": sibling_results,
        "available_artifacts": artifacts,
    }

    return xml_compose(fundamental, planner_specific)
```

## 5.9 Context-Driven Failure Modes in Planning

### F1. Parent context missing

If parent result unavailable, planner may under-specify decomposition constraints.

### F2. Sibling context staleness

Planner can overfit or duplicate work if sibling completion state is stale/incomplete.

### F3. Artifact overload

`full` mode may flood context with irrelevant artifacts, reducing planning signal-to-noise ratio.

### F4. Artifact underexposure

`none` mode may force planner to rediscover known constraints implicitly.

## 5.10 Research Hypotheses for Context Policy

### H1: dependency-only artifacts improve decomposition precision

For many tasks, planner quality may improve under `dependencies` vs `full` due to reduced noise.

### H2: sibling summaries reduce redundant subtask generation

Injecting sibling results should reduce duplicate decomposition branches.

### H3: depth-aware context truncation improves scalability

At deeper levels, aggressive context pruning may reduce token overhead without harming local plan quality.

## 5.11 Suggested Experimental Design

### Factor grid

- artifact mode: `none`, `dependencies`, `subtask`, `full`
- planner prompt variant: baseline vs structure-constrained
- depth range: shallow vs deep decomposition tasks

### Metrics

- subtask redundancy index
- dependency graph validity score
- plan-to-execution repair rate (invalid deps filtered)
- token cost per successful plan
- downstream retries/failures

## 5.12 Protocol Hardening Suggestions

1. Add planner-context checksum metadata in module history for forensic reproducibility.
2. Add maximum artifact count/token cap per planner context.
3. Add relevance scoring before artifact injection in `full` mode.
4. Add explicit sibling-state freshness markers to reduce stale-context risk.

## 5.13 Implementation Anchors

- context orchestrator: `src/roma_dspy/core/context/manager.py`
- context models/XML serialization: `src/roma_dspy/core/context/models.py`
- execution-scoped context: `src/roma_dspy/core/context/execution_context.py`
- artifact querying: `src/roma_dspy/core/artifacts/query_service.py`

