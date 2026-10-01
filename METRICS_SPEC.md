# Metrics and Accounting Specification v1.1

Revision v1.1 incorporates research decisions D-001 through D-007, D-009, and D-011 through D-031 from `DECISIONS.md`.

## 1. Primary outcome

**Execution accuracy / task success**.

The submitted SQL is executed and compared with the benchmark gold result under the project's deterministic execution-grading rule. No LLM judge is used for the headline accuracy metric.

## 2. Run-level metrics
Every completed episode must record:

- execution_success
- raw_prompt_tokens
- cached_prompt_tokens
- completion_tokens
- total_model_tokens
- derived_usd
- wall_clock_latency_ms
- logical_turn_count
- tool_call_count
- clarification_turn_count
- retry_count (includes retries after malformed non-truncated output, D-026)
- termination_reason
- policy
- tool_space
- seed
- query_id
- experiment_version
- run_mode (`experiment` or `playground`, D-015)

## 3. Turn/step-level trace
Every relevant event must record:

- episode_id
- logical_turn_id
- step_id
- action_type
- tool_name (if applicable)
- model_input_tokens
- model_output_tokens
- tool_input_tokens (transport diagnostic)
- tool_output_tokens (transport diagnostic)
- cached_tokens
- incremental_usd
- step_latency_ms
- cumulative_model_tokens
- remaining_budget
- retrieval_latency_ms (Retrieve-then-Load retrieval step only; logged separately, D-009)
- max_completion_tokens_cap (the pre-call cap applied, D-006)
- action_denied (boolean, D-007)
- denial_reason (telemetry only; never model-facing, D-014)
- finish_reason (per model call, e.g. `length` when the completion cap is reached, D-012)
- planner_side_channel (controller metadata, logged in telemetry under all four policies; never model-visible under any policy, D-024; schema in `POLICY_SPEC.md`, D-022, D-025)
- completion_cap_binding (`budget` or `context`: which limit set `max_completion_tokens`, D-020)
- output_parse_valid (boolean: response parsed as a complete valid action/answer under the strict schema, D-021)
- estimated_incremental_tokens and actual_incremental_tokens (Calibrated-VoI selections only; new model input tokens introduced into the next model-visible state by the selected action, D-022, D-029). `actual_incremental_tokens` is measured from the labelled prompt segments (§5) introduced by the selected action, including deterministic wrapper text directly caused by that insertion; it is not an unrestricted difference between successive full prompts.
- voi_decision (Calibrated-VoI only: the selected consumer action, or `PROCEED_TO_SQL`, D-025, D-030)

## 4. Eight analytic buckets
The trace should support the following decomposition used in the project design:

1. exposure / schema
2. exploration
3. clarification
4. planning / reasoning
5. SQL generation
6. execution results
7. self-correction
8. verification

The research question's three competing consumers are specifically buckets 1–3. Buckets 4–8 are measured stages of the fixed agent.

Bucket assignments fixed by decision:

- Static-All's upfront full database schema → bucket 1, exposure / schema (D-002).
- JIT schema expansion (policy-layer control-plane operation, not an MCP tool) → bucket 1, exposure / schema (D-001).
- Calibrated-VoI confidence comes from the existing fixed planner interaction; there is no separate confidence-elicitation call and therefore no additional bucket for it (D-003).

## 5. Token attribution rule
Prompts are assembled from labelled segments before the LLM call. Each segment is tokenized using the exact tokenizer associated with the experimental model.

The serving layer's aggregate input-token count is reconciled against the locally calculated segment total, allowing only a documented small residual for chat-template / separator overhead.

Unattributable overhead goes to a residual/system-overhead bucket rather than being silently assigned to a consumer.

## 6. No double counting
If a tool returns text that is subsequently included in an LLM prompt:

- the returned text is logged as tool-output transport metadata;
- when that text enters the next model prompt, it is counted as model input once;
- it is not added again as a second model-token charge merely because it crossed the tool boundary.

## 7. Cached tokens
Raw prompt tokens, cached prompt tokens, and completion tokens are recorded separately.

Raw model-token accounting is the primary budget currency. Derived dollar cost must state the pricing convention and must not be presented as a measured vendor invoice when inference is self-hosted.

## 7a. Token budget B (D-005, D-006)
Budget currency:

`budget_used = Σ over all model calls in the episode (raw model input tokens + completion tokens)`

- Cached tokens are logged separately; they are not subtracted from raw input for budget purposes.
- Tool output is charged when it is inserted into a model prompt (consistent with §6).
- User-simulator generation is not separately charged; simulator reply text is charged when it enters the next agent prompt (consistent with §8).
- Retriever compute is not part of B; retrieval latency is logged separately (D-009).

Pre-call cap (D-006, D-012, D-020). Before each model call:

```text
remaining_episode_budget = B − cumulative_raw_tokens − current_input_tokens
context_headroom         = model_context_limit − current_input_tokens
max_completion_tokens    = min(remaining_episode_budget, context_headroom)
```

where `cumulative_raw_tokens` is `budget_used` over all prior model calls in the episode.

- If `remaining_episode_budget <= 0`, the call is not issued and the episode terminates with `termination_reason = budget_exhausted`.
- If `current_input_tokens` exceeds the model context window, the input is not truncated, the call is not issued, and the episode terminates with `termination_reason = context_limit_exceeded`.
- Otherwise, the call is issued with `max_completion_tokens` as above, and `completion_cap_binding` records whether B (`budget`) or the context window (`context`) set the cap.
- There is no minimum completion threshold.
- No silent truncation of input or context.
- No episode may exceed B.

Truncated responses (D-012, D-021). Model actions use a strict deterministic structured output schema.

- If a response hits the completion cap but parses as a complete valid action or answer under that schema, it is processed normally.
- If a response is truncated (`finish_reason = length`) and does not parse as a complete valid action or answer, the episode terminates as an incomplete / truncated response: `budget_exhausted` when `completion_cap_binding = budget`, `context_limit_exceeded` when `completion_cap_binding = context`.
- Validity is decided only by the deterministic schema parse. No LLM judge and no human interpretation.

Malformed non-truncated responses (D-026):

- A response that is not truncated but fails the strict structured output schema is retried if the retry cap has not been reached.
- The model call that produced it is charged normally.
- The retry is not a logical turn.
- Once the retry cap is exhausted, the episode terminates with `termination_reason = malformed_output`.
- No LLM judge and no human interpretation.

Denied actions (D-007):

- A denied action consumes zero action/tool cost and does not create a logical turn.
- The model-facing denial message is `ACTION_NOT_PERMITTED: The requested action is unavailable in the current state.` for every policy (D-014). It is charged normally when it enters a later model prompt.
- The detailed denial reason is recorded in telemetry only.

## 8. User simulator
The user simulator is the specified function-driven server capability. Its returned text is environment output and is counted when incorporated into a subsequent model prompt.

Do not fabricate simulator generation-token charges unless the implementation actually uses an LLM-backed simulator and the experiment specification is amended.

## 9. Logical turn definition (D-004)
A logical turn ends when an externally observable allocation action / tool interaction or user-simulator reply is completed and the resulting state is returned.

Counting rules (D-011):

| Event | Logical turn |
|---|---|
| Static-All initial schema exposure (initialization) | 0 |
| Retrieve-then-Load initial retrieval / load (initialization) | 0 |
| JIT initial name / summary exposure (initialization) | 0 |
| JIT schema expansion (policy-layer control plane) | 1 |
| `list_tables` | 1 |
| `describe_table` | 1 |
| `sample_values` | 1 |
| `search_knowledge_base` | 1 |
| `get_column_description` | 1 |
| `ask_user` + simulator response (together) | 1 |
| `execute_sql` — every actual interaction, whether on the normal path, by verification, or during an internal retry | 1 |
| `submit_answer` | 1 |
| Planner / SQL-writer / verifier internal LLM calls (including the calls that cause verification or a retry) | 0 |
| Internal retries (the retry itself; any `execute_sql` it issues counts as above) | 0 |
| Denied actions | 0 |

Initialization-time exposure is not a logical turn (D-019). Logical turn 1 begins with the first actual allocation / tool interaction or user-simulator interaction after initialization. Every actual `execute_sql` interaction counts as exactly one logical turn (D-018).

Low-level instrumentation events do not become separate logical turns. The same counting rules apply under all four policies.

## 10. Termination
An episode terminates on the first of:

- valid answer submission;
- maximum logical-turn cap reached;
- token budget B exhausted, `termination_reason = budget_exhausted` (including `remaining_episode_budget <= 0` before a call, and an unparseable response truncated by the budget cap, §7a);
- model context window exceeded, `termination_reason = context_limit_exceeded` (input exceeds the context window before a call, or an unparseable response truncated by the context cap, §7a; D-020). This is distinct from `budget_exhausted`;
- malformed non-truncated output after the retry cap is exhausted, `termination_reason = malformed_output` (D-026);
- unrecoverable runtime failure, which must be logged distinctly from a wrong answer.

A wrong submitted answer and a turn-cap termination are both failures for execution accuracy, but must remain different `termination_reason` values.

## 11. Efficiency metrics
Compute/report:

- accuracy per dollar;
- accuracy-vs-cost curves;
- area under the accuracy-cost curve;
- Pareto-dominance count;
- retrieval diagnostics: gold-tool recall@k, precision, MRR;
- reliability across three seeds, including bootstrap confidence intervals and pass^k where applicable.

## 12. Attribution sanity checks
Before main experiments begin, create synthetic prompts where the expected segment token counts are known. The accounting system must reconcile before policy logic is considered validated.

## 13. Run modes and ledger separation (D-015)
- `experiment` mode: controlled, frozen, and reproducible. Writes only to the controlled experiment ledger (`results/`).
- `playground` mode: live episodes launched from the Streamlit playground. Controls may vary policy, tool space, token budget, task, and seed. Writes only to a separate playground-results location.
- Playground records never enter, are never merged into, and are never summarised with the controlled experiment ledger.
- Every ledger record carries its `run_mode`.
