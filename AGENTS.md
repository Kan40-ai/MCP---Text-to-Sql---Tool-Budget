# Codex Repository Instructions

## Mission
Implement the MCP Text-to-SQL tool-budget experiment strictly according to the research contracts in this directory.

Read these files before making substantive changes:

1. PROJECT_SPEC.md
2. ARCHITECTURE.md
3. POLICY_SPEC.md
4. METRICS_SPEC.md
5. TEST_PLAN.md
6. EXPERIMENT_MATRIX.yaml
7. DECISIONS.md (research decision log; entries are binding)

## Highest-priority rule
Do not invent research semantics.

Codex may choose implementation details, module boundaries, libraries, error handling, and test structure, but must not independently decide:

- what a policy means;
- what counts as budget B;
- what counts as a logical turn;
- what constitutes a research variable/control;
- how tokens are attributed;
- what constitutes task success;
- what model is used in the main experiment.

If a research-semantic value is unspecified, stop and mark it `PILOT_LOCK_REQUIRED` or ask the researcher.

## Workflow
Use staged agentic engineering:

1. Inspect the repository.
2. Produce a short implementation plan before a major code change.
3. Implement the smallest testable vertical slice.
4. Run relevant tests.
5. Inspect failures/logs.
6. Fix root causes.
7. Re-run tests.
8. Produce an evidence summary.
9. Freeze only after the Definition of Done gate passes.

Do not build the full 3,600-episode matrix before the one-query vertical slice and accounting gate pass.

## Development rules
- Keep the LLM backend swappable between MockLLM and the fixed experimental backend.
- Keep configuration externalized; do not hard-code experiment grids into application logic.
- Use deterministic, explicit IDs for episode/query/policy/tool-space/seed.
- Make all experiment runs checkpointable and resumable.
- Preserve raw per-call telemetry in addition to aggregated metrics.
- Separate research metrics from transport/debug metrics.
- Never silently swallow errors or convert runtime failures into task failures.
- Never silently truncate context to make a condition fit.
- Do not add explicit ReAct as a new experimental factor.

## Testing gate
Before declaring work complete, run the relevant tests and report the evidence. The project cannot be declared ready for pilot until the gates in TEST_PLAN.md pass.

## Research integrity
Never modify benchmark tasks, labels, prompts, model configuration, policy definitions, or evaluator logic after seeing main-experiment results without creating a new experiment version and documenting the change.
