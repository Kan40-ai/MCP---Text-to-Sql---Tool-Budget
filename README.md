# Start Here — MCP Text-to-SQL Agentic Engineering Contracts

This folder contains the research/engineering contracts for the project.

## Current status
- Specifications only. No implementation code, tests, configs, or UI exist yet.
- Architecture: a fixed Planner / SQL Writer / Verifier orchestrator, with an allocation-policy gate (Static-All, Retrieve-then-Load, JIT / Progressive, Calibrated-VoI) over three budget consumers (schema exposure, exploration, clarification), against a common MCP layer (database, knowledge base, user simulator) plus `submit_answer`. A planned Streamlit playground is a non-experimental layer whose results never enter the experiment ledger. See `ARCHITECTURE.md`.
- Decision log: `DECISIONS.md` records binding human-approved decisions (D-001 onward) and open items. Coding agents may record decisions the researcher has explicitly provided and approved, but may not invent, modify, or self-approve them.
- Still open before main experiments: the `PILOT_LOCK_REQUIRED` parameters in `EXPERIMENT_MATRIX.yaml`, and the open items listed in `DECISIONS.md` (including the Calibrated-VoI calibration protocol, O-014, which blocks any real calibrator fitting).

## Files
- `PROJECT_SPEC.md` — overall research scope and invariants
- `ARCHITECTURE.md` — runtime architecture and component boundaries
- `POLICY_SPEC.md` — exact semantics of the four policies
- `METRICS_SPEC.md` — token, cost, turn, and trace accounting
- `TEST_PLAN.md` — tests and Definition of Done
- `EXPERIMENT_MATRIX.yaml` — experimental grid and pilot-lock parameters
- `DECISIONS.md` — research decision log (binding decisions and open items)
- `AGENTS.md` — Codex-specific entry instructions
- `CLAUDE.md` — Claude Code-specific entry instructions

## How to use
Copy this directory's contents into the root of the implementation repository.

For Codex, `AGENTS.md` is the coding-agent instruction entrypoint.
For Claude Code, `CLAUDE.md` is the corresponding instruction entrypoint.

The other seven files (six specifications plus `DECISIONS.md`) are the same research contracts regardless of coding agent.

## First task for the coding agent
Do not start by implementing the entire system.

First:
1. inspect the repository;
2. identify missing dependencies/data;
3. produce a plan for the one-query vertical slice;
4. list any unresolved `PILOT_LOCK_REQUIRED` items;
5. implement only the smallest slice needed to reach the first test gate.
