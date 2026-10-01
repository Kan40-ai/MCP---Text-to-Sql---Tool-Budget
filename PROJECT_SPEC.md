# MCP Text-to-SQL Tool-Budget Experiment — Project Specification v1.0

## 1. Purpose
This repository implements a controlled empirical study of cost-aware orchestration in an MCP-mediated Text-to-SQL analytics agent.

The research question is:

> Given a fixed per-episode token budget B for an MCP-mediated analytics agent, how should the orchestrator allocate that budget across (a) tool-schema exposure, (b) database/knowledge-base exploration, and (c) user clarification, and what accuracy-versus-cost frontier do different allocation policies trace?

The contribution is a controlled measurement study, not a new foundation model, a new benchmark, or a second agent topology.

## 2. Experimental invariants
The following are frozen across the main experiment:

- One backbone model: Qwen3-8B.
- One planner / SQL writer / verifier topology.
- Same prompts and system instructions.
- Same database / knowledge-base / user-simulator environment.
- Same eight real MCP tools in every condition.
- Same task set.
- Same per-episode budget B.
- Same logical-turn cap.
- Same retry cap (maximum three retries).
- Same execution-based grading.
- Same evaluation code.
- Only the allocation policy changes across the four policy conditions.

Any change to a research invariant after the main experiment begins invalidates comparability with prior runs and must be recorded as a new experiment version.

## 3. Four policy conditions
1. Static-All
2. Retrieve-then-Load
3. JIT / Progressive
4. Calibrated-VoI

The common MCP layer is available to all four policies. Calibrated-VoI is distinguished by dynamically reallocating budget among the three consumers inside the episode; MCP availability itself is not a policy-specific feature.

## 4. Three budget consumers
- Consumer (a): tool-schema exposure
- Consumer (b): database / knowledge-base exploration
- Consumer (c): user clarification

These are competing consumers of the same per-episode budget B.

SQL generation, verification, planning/reasoning, execution results, and self-correction are measured stages of the fixed agent. They are not additional policy conditions.

## 5. Tool-space conditions
The agent-visible registry is manipulated at three sizes:

- 25 tools = 8 real + 17 distractors
- 100 tools = 8 real + 92 distractors
- 400 tools = 8 real + 392 distractors

Distractors are stratified into near, mid, and far bands. The pools are nested, mean schema length is normalized across sizes, and tool position is shuffled by seed to control position bias.

## 6. Dataset
Primary environment: BIRD-Interact Mini-Interact.

Main set: a frozen 100-task subsample stratified by ambiguity and SQL complexity.

Controls:
- BIRD dev cleaned 1106 split
- Spider 1.0 dev

A 30-task gold-label hand audit is required before main policy evaluation.

## 7. Main experiment matrix
100 tasks × 4 policies × 3 tool-space sizes × 3 seeds = 3,600 independent episodes.

The full matrix must be generated from configuration rather than hard-coded into the runner.

## 8. Success criteria
The implementation is ready for scale only after:

- one end-to-end query succeeds;
- all four policies can execute against the same MCP environment;
- per-step token accounting reconciles to serving-layer usage within the documented tolerance;
- deterministic execution grading works;
- one experimental cell is reproducible from one command;
- checkpoints survive runtime interruption;
- the experiment runner can resume without duplicating completed episodes.

## 9. Out of scope
- Fine-tuning, LoRA, RL, or other policy training.
- A second backbone model in the main comparison.
- A second orchestrator topology.
- Spider 2.0-Snow leaderboard pursuit.
- Multi-dialect / cloud-warehouse execution.
- Human-subject evaluation.
- Building a new benchmark.
- Explicit ReAct as an experimental factor.

## 10. Pilot-lock parameters
The following must be empirically selected during the environment/measurement pilot and frozen before the main experiment:

- exact model precision / quantization compatible with the selected GPU;
- numeric value of B;
- numeric maximum logical-turn cap;
- retriever implementation and k;
- VOI calibration / decision thresholds;
- exact latency measurement convention.

The coding agent must not invent these values merely to complete the implementation. It should surface the parameter as `PILOT_LOCK_REQUIRED` and provide the configuration hook.
