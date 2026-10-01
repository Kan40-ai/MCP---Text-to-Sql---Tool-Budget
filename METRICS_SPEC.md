# Metrics and Accounting Specification v1.0

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
- retry_count
- termination_reason
- policy
- tool_space
- seed
- query_id
- experiment_version

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

## 8. User simulator
The user simulator is the specified function-driven server capability. Its returned text is environment output and is counted when incorporated into a subsequent model prompt.

Do not fabricate simulator generation-token charges unless the implementation actually uses an LLM-backed simulator and the experiment specification is amended.

## 9. Logical turn definition
A logical turn is one benchmark-level interaction cycle ending after the active action/tool interaction or user-simulator response has completed and the agent receives the resulting state.

Low-level instrumentation events inside that cycle do not automatically become separate logical turns.

## 10. Termination
An episode terminates on the first of:

- valid answer submission;
- maximum logical-turn cap reached;
- token budget B exhausted;
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
