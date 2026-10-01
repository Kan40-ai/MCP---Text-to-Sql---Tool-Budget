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

### Malformed-output tests (D-026)
- a non-truncated response that fails the strict schema triggers a retry while the retry cap has not been reached;
- the malformed model call is charged normally, and the retry does not increment the logical-turn counter;
- after the retry cap is exhausted, the episode terminates with `termination_reason = malformed_output` (and no synonym);
- validity is decided only by the deterministic schema parser.

### Completion-cap tests (D-006, D-012, D-020, D-021)
- on every model call, the `max_completion_tokens` passed to the backend equals `min(B − cumulative_raw_tokens − current_input_tokens, model_context_limit − current_input_tokens)`;
- `completion_cap_binding` is `budget` when B sets the cap and `context` when the context window sets it;
- when `remaining_episode_budget <= 0`, no model call is issued and the episode terminates with `budget_exhausted`;
- when `current_input_tokens` exceeds the context window, no model call is issued, the input is not truncated, and the episode terminates with `context_limit_exceeded`;
- no minimum completion threshold is applied (e.g. a cap of 1 issues a call with `max_completion_tokens = 1`);
- a response truncated at the cap that parses as a complete valid action/answer under the strict schema is processed normally;
- a truncated response that does not parse terminates with `budget_exhausted` (budget-bound) or `context_limit_exceeded` (context-bound), with `finish_reason = length` recorded;
- validity is determined only by the deterministic schema parser (fixture responses: complete, truncated-but-complete, truncated-incomplete);
- across all test episodes, cumulative raw tokens never exceed B, and no prompt is silently truncated.

### Retrieve-then-Load out-of-set ban (D-008, D-009)
- retrieval is invoked exactly once per episode;
- a call to any tool (real or distractor) outside the retrieved set is denied and never reaches an MCP server;
- the out-of-set denial follows the denied-action rules below;
- retrieval latency is recorded separately and contributes zero tokens to B.

### Denied-action tests (D-007, D-014)
- a denied action adds zero action/tool cost and does not increment the logical-turn counter;
- the model-facing message is exactly `ACTION_NOT_PERMITTED: The requested action is unavailable in the current state.` under every policy;
- the detailed denial reason appears in telemetry and never in any model prompt;
- the denial message is charged as model input when it enters a later prompt, and is not charged otherwise.

### Logical-turn counting tests (D-004, D-011)
- JIT schema expansion, `list_tables`, `describe_table`, `sample_values`, `search_knowledge_base`, `get_column_description`, `execute_sql`, and `submit_answer` each increment the turn counter by exactly one;
- `ask_user` plus its simulator response together increment the counter by exactly one;
- every actual `execute_sql` interaction increments the counter by exactly one, whether on the normal path, by verification, or during an internal retry (D-018);
- planner / SQL-writer / verifier internal LLM calls (including those that cause verification or a retry), internal retries themselves, and denied actions increment the counter by zero;
- initialization-time exposure increments the counter by zero under Static-All, Retrieve-then-Load, and JIT; logical turn 1 is the first actual interaction after initialization (D-019);
- the same scripted event sequence yields the same turn count under all four policies;
- reaching the turn cap terminates with a turn-cap `termination_reason` distinct from `budget_exhausted`.

### Planner side-channel tests (D-003, D-013, D-017, D-023, D-024)
- the same planner call, planner prompt template / instruction contract, and structured output schema are used under all four policies, and the side-channel signal is logged under all four;
- under all four policies, including Calibrated-VoI, no side-channel content appears in any model prompt (checked against the labelled prompt segments);
- only Calibrated-VoI reads the signal for allocation;
- no policy issues a confidence-elicitation model call; the set of model-call types is identical across policies.

### Calibrated-VoI computation tests (D-022, D-025, D-027 through D-031)
For fixed synthetic side-channel outputs, a mock calibrator (a fixed synthetic monotone mapping, not a fitted model), a fixed threshold value, and a fixed cost lookup:
- `Gain(a) = max(0, Cal(p_success_after(a)) − Cal(p_success_now))` and `VoI(a) = Gain(a) / estimated_incremental_tokens(a)` match hand-computed values;
- the same single calibrator is applied to `p_success_now` and every `p_success_after`;
- candidate actions are restricted to the `schema`, `exploration`, and `clarification` consumers; a `commit` candidate is rejected by the schema;
- the highest-VoI permitted consumer action is selected when its VoI is at or above the threshold;
- when the maximum permitted consumer VoI is below the threshold, no consumer action is selected and the episode proceeds to the fixed SQL path;
- ties are broken by: higher VoI, then lower estimated incremental token cost, then consumer priority `schema` > `exploration` > `clarification`, then lexicographic canonical action identifier within a consumer (D-028);
- the selection is identical when candidate actions are presented in shuffled order or under different seeds;
- when no permitted consumer action exists, the policy returns `PROCEED_TO_SQL`, the fixed SQL path continues, and the episode is not terminated for that reason (D-030);
- `estimated_incremental_tokens` excludes the selecting model call, future SQL-generation and verifier completions, and retriever compute; `actual_incremental_tokens` counts only the new input tokens the action's result introduced into the next model-visible state, measured from the labelled prompt segments introduced by that action plus their deterministic wrapper text, and differs from a naive full-prompt difference when unrelated segments also change between prompts (D-029);
- no real isotonic-fitting procedure exists or is invoked, and no real calibration artifact is loaded (D-027, D-031);
- denied / non-permitted candidates are never selected;
- estimated and actual incremental token costs are both logged for each selection;
- the cost estimate is a lookup (no fitted predictor is invoked);
- side-channel output that fails the strict schema is handled per D-021, never by interpretation.

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

## 7a. Playground / experiment ledger separation (D-015)
- a playground episode writes only to the playground-results location;
- after playground runs, the controlled experiment ledger is byte-identical to its prior state;
- the experiment runner and summary statistics never read from the playground-results location;
- every ledger record carries `run_mode`, and the controlled experiment ledger contains no `playground` records;
- the research engine imports and its tests pass with the UI package absent.

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
