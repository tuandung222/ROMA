# Advanced ROMA Planning Notebook Suite

Execution order:

1. `01_ROMA_Agent_Architecture_and_Execution_Model.ipynb`
2. `02_Planner_Introspection_and_Plan_Quality_Lab.ipynb`
3. `03_End_to_End_Orchestration_Trace_and_Checkpoint_Analysis.ipynb`

What this suite covers:

- Agent architecture and runtime state machine in ROMA
- Planner contract (`SubTask`, `dependencies_graph`) and conversion into DAG dependencies
- End-to-end orchestration trace with checkpoint-level evidence

Live inference toggles (inside notebook code cells):

- Notebook 2: `RUN_LIVE_PLANNER = True`
- Notebook 3: `RUN_LIVE_EXECUTION = True`
- Notebook 3 comparison: `RUN_MODE_COMPARISON = True`

Requirements:

- Python environment with ROMA installed (recommended: conda env `vllm`)
- `OPENAI_API_KEY` set for live model calls
- Run notebooks from repository root (`/Users/admin/TuanDung/repos/ROMA`) for stable path resolution
