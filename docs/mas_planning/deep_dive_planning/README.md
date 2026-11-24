# Deep-Dive Planning and Orchestration Dossier

This folder contains a research-grade technical dossier on ROMA's planning/orchestration mechanisms for multi-agent systems (MAS).

Audience:
- AI researchers
- MAS framework architects
- engineers designing planning policies/protocols

Scope:
- implementation-grounded analysis of planning runtime behavior
- protocol-level contracts (inputs/outputs/state transitions)
- orchestration policy trade-offs (recursive vs event-driven)
- resilience and checkpoint semantics
- observability and evaluation methodology

Document map:

1. `00_research_scope_and_methodology.md`
   - research framing and inspection methodology
   - implementation boundaries and assumptions

2. `01_formal_system_model.md`
   - formal state-space model of tasks, DAGs, and transitions
   - invariants and correctness properties

3. `02_planning_protocol_and_contracts.md`
   - planner protocol, data contracts, dependency normalization
   - context protocol and index-to-task materialization

4. `03_orchestration_policies_and_scheduler.md`
   - recursive state-machine policy vs event-driven policy
   - scheduler queueing, priority, retry, and completion semantics

5. `04_failure_recovery_checkpoint_protocol.md`
   - checkpoint triggers, snapshot schema, recovery plan semantics
   - partial/full/selective recovery behavior

6. `05_context_artifact_protocol.md`
   - cross-agent context contract and artifact injection modes
   - information-flow implications for planner quality

7. `06_observability_eval_and_research_agenda.md`
   - trace model, metrics, diagnostics pipeline
   - suggested experimental agenda for planning/orchestration research

Primary code anchors:
- `src/roma_dspy/core/engine/solve.py`
- `src/roma_dspy/core/engine/runtime.py`
- `src/roma_dspy/core/engine/dag.py`
- `src/roma_dspy/core/engine/event_loop.py`
- `src/roma_dspy/core/engine/scheduler.py`
- `src/roma_dspy/core/context/manager.py`
- `src/roma_dspy/resilience/checkpoint_manager.py`
- `src/roma_dspy/core/signatures/signatures.py`
- `src/roma_dspy/core/signatures/base_models/subtask.py`

