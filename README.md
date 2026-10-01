# Start Here — MCP Text-to-SQL Agentic Engineering Contracts

This folder contains the frozen research/engineering contracts for the project.

## Files
- `PROJECT_SPEC.md` — overall research scope and invariants
- `ARCHITECTURE.md` — runtime architecture and component boundaries
- `POLICY_SPEC.md` — exact semantics of the four policies
- `METRICS_SPEC.md` — token, cost, turn, and trace accounting
- `TEST_PLAN.md` — tests and Definition of Done
- `EXPERIMENT_MATRIX.yaml` — experimental grid and pilot-lock parameters
- `AGENTS.md` — Codex-specific entry instructions
- `CLAUDE.md` — Claude Code-specific entry instructions

## How to use
Copy this directory's contents into the root of the implementation repository.

For Codex, `AGENTS.md` is the coding-agent instruction entrypoint.
For Claude Code, `CLAUDE.md` is the corresponding instruction entrypoint.

The other six files are the same research contracts regardless of coding agent.

## First task for the coding agent
Do not start by implementing the entire system.

First:
1. inspect the repository;
2. identify missing dependencies/data;
3. produce a plan for the one-query vertical slice;
4. list any unresolved `PILOT_LOCK_REQUIRED` items;
5. implement only the smallest slice needed to reach the first test gate.
