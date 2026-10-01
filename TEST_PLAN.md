# Test Plan and Definition of Done v1.0

## 1. Test philosophy
No large-scale experiment begins until the implementation has passed unit, integration, accounting, policy-isolation, and reproducibility gates.

The coding agent must provide evidence for every gate.

## 2. Unit tests
### MCP tools
- each real tool validates inputs and returns schema-conformant output;
- `execute_sql` permits only read-only statements;
- invalid tool calls fail predictably;
- distractor tools conform to the same registry representation.

### Policy unit tests
For fixed synthetic states:
- Static-All exposes all schemas at initialization;
- Retrieve-then-Load retrieves exactly k once;
- JIT exposes names/summaries initially and deeper schema only when requested;
- Calibrated-VoI evaluates all three consumer candidates and can choose different consumers on successive states.

### Budget tests
- remaining budget never becomes negative;
- denied actions do not silently consume budget;
- retries are recharged according to the same accounting rule;
- budget exhaustion terminates correctly.

## 3. Integration tests
Minimum vertical slice:

```text
1 query
× 1 tool-space
× 4 policies
× 1 seed
```

Each policy must complete through:

`query -> planner -> policy -> MCP -> SQL -> execution -> verification -> submit_answer`

or terminate with a clearly logged reason.

## 4. Accounting tests
For every LLM call:

`sum(labelled prompt segments) + documented residual ≈ server-reported input tokens`

Also verify:

`sum(call-level tokens) = episode total tokens`

within the documented tolerance.

## 5. Policy-isolation test
Run the same query/environment/seed under each policy. Confirm that configuration diff shows only the policy identifier and its policy-controlled behavior.

The coding agent must not silently alter prompts, model settings, MCP interfaces, database state, or evaluator code per policy.

## 6. Reproducibility test
Run one cell twice from a clean environment using identical configuration. Compare:

- query ID
- policy
- tool-space
- seed
- tool registry composition
- model configuration
- ledger schema
- execution result

Any material difference must be explained before scale-up.

## 7. Resume test
Intentionally terminate a run batch midway. Restart from checkpoint. Verify:

- completed episode IDs are not duplicated;
- incomplete episodes are resumed;
- ledger remains append-safe;
- summary statistics use each episode exactly once.

## 8. Four explicit smoke scenarios
### S1 — Directly solvable, low ambiguity
The agent should be able to solve without clarification.

### S2 — Requires environment exploration
The agent needs table/column/value discovery.

### S3 — Requires user clarification
The agent should receive a meaningful simulated clarification before final SQL.

### S4 — Requires recovery
An intentionally controlled SQL error or mismatch should trigger self-correction and show recharged cost.

## 9. 400-tool safety test
Before allowing the 400-tool condition to enter the main experiment:

- measure actual schema token count;
- measure maximum prompt size;
- verify model context compatibility;
- record headroom;
- confirm the condition is not silently truncated.

If the context ceiling binds, stop and escalate to the research owner rather than changing the model or silently truncating the condition.

## 10. Definition of Done for implementation
The project implementation is considered ready for pilot when all are true:

- all required servers start from one documented command;
- all four policies run through the same agent/orchestrator code path;
- token accounting reconciles;
- execution grading is deterministic;
- per-episode and per-step ledgers are complete;
- checkpoints are durable;
- one cell reproduces from one command;
- tests pass in a clean environment;
- no unresolved `BLOCKER` or `PILOT_LOCK_REQUIRED` item remains hidden.

## 11. Coding-agent reporting requirement
After each implementation stage, the coding agent must report:

1. files changed;
2. tests run;
3. tests passed/failed;
4. observed evidence;
5. assumptions introduced;
6. open blockers;
7. whether the stage is safe to freeze.
