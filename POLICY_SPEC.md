# Policy Specification v1.0

## Policy interface
Each policy implements a common interface so that the orchestrator is policy-agnostic.

Conceptually:

```python
policy.initialize(episode_state)
policy.decide(current_state, candidate_actions, remaining_budget)
policy.update(observation)
```

Exact code is an implementation choice; the research semantics below are not.

## Policy 1 — Static-All

### Definition
At episode start, expose all visible tool schemas and the full database schema according to the experimental registry condition.

### Research role
Naive maximal-exposure baseline / cost ceiling.

### Important behavior
Static-All is not required to use exploration or clarification. Those may occur if the fixed agent requests them and the policy permits them under the same common experimental rules.

### Required invariant
Initial exposure is fixed by the registry size; it is not selectively retrieved.

## Policy 2 — Retrieve-then-Load

### Definition
Before execution, retrieve the top-k candidate tools once and load those schemas.

### Parameters
- Retriever: selected on validation data.
- k: selected on validation data and frozen before the main experiment.

### Research role
Cited retrieval-gated baseline.

### Important behavior
This policy is not allowed to dynamically rerun retrieval at every turn unless that is explicitly added as a new policy definition. Exploration and user clarification remain governed by the common agent/environment rules.

## Policy 3 — JIT / Progressive

### Definition
Expose tool names and one-line summaries initially. Full tool schemas and deeper database/knowledge-base details are revealed only when requested/needed under the policy contract.

### Research role
Incremental/lazy exposure condition.

### Important behavior
JIT is not a requirement to use every information route. A route can remain unused when it is unnecessary.

## Policy 4 — Calibrated-VoI

### Definition
At each policy decision point, score candidate actions across all three consumers:

1. schema exposure;
2. environment exploration;
3. user clarification.

The policy allocates the next budget-consuming action to the option with the highest expected accuracy gain per incremental token cost, subject to remaining budget and action validity.

### Dynamic behavior
The selected consumer may repeat. Example:

```text
Explore -> Explore -> Schema -> Ask -> Explore -> SQL
```

There is no requirement to use all three consumers.

### Calibration
The current research design uses verbalised confidence followed by isotonic calibration on a validation split disjoint from the frozen 100-task main set. Calibration and decision thresholds are frozen before the main experiment.

## Common rules across all four policies

- Same model.
- Same prompts.
- Same orchestrator.
- Same MCP servers.
- Same real tools.
- Same distractor pool for a given tool-space/seed.
- Same query.
- Same B.
- Same turn cap.
- Same retry cap.
- Same execution grader.

## Policy isolation test
A policy switch must be configurable without changing the fixed-agent code path, MCP server definitions, database contents, or evaluation code.
