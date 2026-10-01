# Research Decision Log

This log records research-semantic decisions made by the research owner. Each entry is binding for the experiment version stated. Specification files reference entries by ID.

Rules:
- Entries are append-only. A reversed or amended decision gets a new entry that names the entry it supersedes.
- A decision made after main-experiment results exist requires a new experiment version (see `AGENTS.md`, Research integrity).
- A coding agent may record a decision only when it has been explicitly provided and approved by the human researcher. A coding agent may not invent, modify, or self-approve research decisions (D-016).

Experiment version affected by all entries below: `1.0` (pre-pilot; no main-experiment results exist).

---

## D-001 — JIT schema expansion is a policy-layer control-plane operation
- **Date:** 2026-10-01
- **Decision:** In JIT / Progressive, expanding a tool from name + one-line summary to its full schema is a policy-layer control-plane operation. It is not a ninth MCP tool and does not appear in the agent-visible tool registry.
- **Consumer:** schema exposure (consumer (a), bucket 1).
- **Consequence:** The registry sizes (25 / 100 / 400) and the eight real MCP tools are unchanged by JIT.
- **Specs updated:** `POLICY_SPEC.md`, `ARCHITECTURE.md`, `METRICS_SPEC.md`.

## D-002 — Static-All upfront database schema is charged to schema exposure
- **Date:** 2026-10-01
- **Decision:** The full database schema that Static-All exposes at episode start is charged to the schema-exposure consumer (consumer (a), bucket 1), not to exploration.
- **Specs updated:** `POLICY_SPEC.md`, `METRICS_SPEC.md`.

## D-003 — Calibrated-VoI uses no separate confidence-elicitation call
- **Date:** 2026-10-01
- **Decision:** Calibrated-VoI must not introduce a separate LLM call to elicit confidence. Confidence / uncertainty information must come from the existing fixed planner interaction and is calibrated with the planned isotonic regression.
- **Consequence:** Because prompts are invariant across policies, the planner interaction that yields confidence is the same for all four policies; only Calibrated-VoI consumes the calibrated value.
- **Specs updated:** `POLICY_SPEC.md`, `METRICS_SPEC.md`.

## D-004 — Logical turn definition
- **Date:** 2026-10-01
- **Decision:** A logical turn ends when an externally observable allocation action / tool interaction or user-simulator reply is completed and the resulting state is returned. Internal LLM calls, verifier calls, and internal retries do not automatically create new logical turns.
- **Supersedes:** the wording of `METRICS_SPEC.md` §9 v1.0 (refinement, not reversal).
- **Specs updated:** `METRICS_SPEC.md`, `ARCHITECTURE.md`.

## D-005 — Token budget B currency
- **Date:** 2026-10-01
- **Decision:** B is measured as cumulative raw model input tokens + completion tokens across all model calls in the episode.
  - Cached tokens are logged separately.
  - Tool output is charged when it is inserted into a model prompt.
  - User-simulator generation is not separately charged; simulator reply text is charged when it enters the next agent prompt.
- **Specs updated:** `METRICS_SPEC.md`.

## D-006 — Pre-call completion cap; no episode may exceed B
- **Date:** 2026-10-01
- **Decision:** Before every model call, the runner caps max completion tokens to the remaining episode budget after accounting for the current input. No episode may exceed B.
- **Consequence:** If the current input alone leaves no completion allowance, the call is not issued and the episode terminates with token-budget exhaustion.
- **Specs updated:** `METRICS_SPEC.md`, `ARCHITECTURE.md`.

## D-007 — Denied actions
- **Date:** 2026-10-01
- **Decision:** A denied action consumes zero action/tool cost and does not create a logical turn. Any resulting denial message that enters a later model prompt is charged normally.
- **Consequence:** The model call that proposed the denied action is still charged under D-005, because it is a model call.
- **Specs updated:** `POLICY_SPEC.md`, `METRICS_SPEC.md`.

## D-008 — Retrieve-then-Load restricts callability to the retrieved set
- **Date:** 2026-10-01
- **Decision:** Retrieve-then-Load retrieves exactly once. Tools outside the retrieved set are not callable for the remainder of the episode.
- **Specs updated:** `POLICY_SPEC.md`.

## D-009 — Retriever compute is outside the model-token budget
- **Date:** 2026-10-01
- **Decision:** Retriever compute is not part of the model-token budget B. Retrieval latency is logged separately.
- **Specs updated:** `POLICY_SPEC.md`, `METRICS_SPEC.md`.

## D-010 — Streamlit UI is a non-experimental observability / playground layer
- **Date:** 2026-10-01
- **Decision:** A planned Streamlit UI is added as a non-experimental observability / playground layer. It must not alter the research engine or experimental semantics.
- **Specs updated:** `ARCHITECTURE.md`.

---

## D-011 — Logical-turn counting rules
- **Date:** 2026-10-01
- **Refines:** D-004. **Resolves:** O-001.
- **Decision:** The following each count as exactly one logical turn:
  - JIT schema expansion (policy-layer control-plane operation, D-001);
  - `list_tables`;
  - `describe_table`;
  - `sample_values`;
  - `search_knowledge_base`;
  - `get_column_description`;
  - `ask_user` together with the simulator response (one turn);
  - `execute_sql`;
  - `submit_answer`.
- The following are not logical turns:
  - planner / SQL-writer / verifier internal LLM calls;
  - internal retries;
  - denied actions.
- **Specs updated:** `METRICS_SPEC.md`, `TEST_PLAN.md`.

## D-012 — Completion-cap procedure and truncation handling
- **Date:** 2026-10-01
- **Refines:** D-006. **Resolves:** O-002.
- **Decision:** Before each model call:
  - `remaining = B − cumulative_raw_tokens − current_input_tokens`;
  - if `remaining <= 0`, do not issue the call; terminate with `budget_exhausted`;
  - otherwise `max_completion_tokens = remaining`.
- There is no minimum completion threshold.
- If generation reaches the cap and is truncated, record `finish_reason = length` and terminate with `budget_exhausted`, unless the emitted response is already a valid completed action/answer.
- **Specs updated:** `METRICS_SPEC.md`, `ARCHITECTURE.md`, `TEST_PLAN.md`.

## D-013 — Planner uncertainty signal for Calibrated-VoI
- **Date:** 2026-10-01
- **Refines:** D-003. **Partially resolves:** O-003 (remaining part is O-006).
- **Decision:**
  - The fixed planner interaction exists under all four policies.
  - It emits structured uncertainty / action-value information that all four policies receive; only Calibrated-VoI uses it for allocation.
  - There is no separate confidence-elicitation LLM call.
  - The exact structured fields and the mapping from planner estimates to calibrated expected accuracy gain per token must be documented and must not introduce any fitted model beyond the planned isotonic calibration.
- **Specs updated:** `POLICY_SPEC.md`, `ARCHITECTURE.md`, `METRICS_SPEC.md`, `TEST_PLAN.md`.

## D-014 — Denial message
- **Date:** 2026-10-01
- **Refines:** D-007. **Resolves:** O-004.
- **Decision:** The model-facing denial message is identical for all policies:

  `ACTION_NOT_PERMITTED: The requested action is unavailable in the current state.`

  The detailed denial reason is telemetry only and never enters a model prompt.
- **Specs updated:** `POLICY_SPEC.md`, `METRICS_SPEC.md`, `TEST_PLAN.md`.

## D-015 — Playground run mode and ledger separation
- **Date:** 2026-10-01
- **Refines:** D-010. **Resolves:** O-005.
- **Decision:**
  - The Streamlit playground may run live episodes.
  - Playground results are written to a separate playground-results location and never enter the controlled experiment ledger.
  - Playground controls may vary policy, tool space, token budget, task, and seed.
  - Controlled experiment mode remains frozen and reproducible.
  - The UI must not be a dependency of the research engine.
- **Specs updated:** `ARCHITECTURE.md`, `METRICS_SPEC.md`, `TEST_PLAN.md`.

## D-016 — Decision-log governance
- **Date:** 2026-10-01
- **Decision:** A coding agent may record a decision that has been explicitly provided and approved by the human researcher. A coding agent may not invent, modify, or self-approve research decisions.
- **Supersedes:** the original header rule "Coding agents must not add entries on their own authority."
- **Specs updated:** this file's header rules.

## D-017 — Planner signal is a policy-controller side channel
- **Date:** 2026-10-01
- **Supersedes in part:** D-013 (the phrase "that all four policies receive").
- **Decision:**
  - The fixed planner generates the same structured uncertainty / action-value signal under all four policies.
  - The signal is a policy-controller side channel. It is NOT injected into the model-visible state for Static-All, Retrieve-then-Load, or JIT / Progressive.
  - Only Calibrated-VoI consumes the signal for allocation.
  - The same planner call, prompt, and output schema remain part of the fixed orchestrator under all four policies.
- **Specs updated:** `POLICY_SPEC.md`, `ARCHITECTURE.md`, `METRICS_SPEC.md`, `TEST_PLAN.md`.

## D-018 — Every actual `execute_sql` interaction is a logical turn
- **Date:** 2026-10-01
- **Refines:** D-011. **Resolves:** O-007.
- **Decision:** Every actual `execute_sql` interaction counts as exactly one logical turn, regardless of whether it is invoked on the normal path, by verification, or during an internal retry. The LLM calls that cause verification / retry do not themselves count as logical turns.
- **Specs updated:** `METRICS_SPEC.md`, `ARCHITECTURE.md`, `TEST_PLAN.md`.

## D-019 — Initialization-time exposure is not a logical turn
- **Date:** 2026-10-01
- **Refines:** D-011. **Resolves:** O-008.
- **Decision:**
  - Static-All initial schema exposure = 0 logical turns.
  - Retrieve-then-Load initial retrieval / load = 0 logical turns.
  - JIT initial name / summary exposure = 0 logical turns.
  - Logical turn 1 begins with the first actual allocation / tool interaction or user-simulator interaction after initialization.
- **Specs updated:** `METRICS_SPEC.md`, `ARCHITECTURE.md`, `TEST_PLAN.md`.

## D-020 — Completion allowance includes the context window
- **Date:** 2026-10-01
- **Refines:** D-012. **Resolves:** O-009.
- **Decision:**
  - A model call's completion allowance is `min(remaining_episode_budget, model_context_limit − current_input_tokens)`.
  - If `current_input_tokens` already exceeds the context window, do not silently truncate; terminate with the distinct termination reason `context_limit_exceeded`.
  - If the context window, rather than B, limits completion length, record that constraint distinctly from `budget_exhausted`.
  - No silent truncation.
- **Specs updated:** `METRICS_SPEC.md`, `ARCHITECTURE.md`, `TEST_PLAN.md`.

## D-021 — Strict structured output; truncated-response validity
- **Date:** 2026-10-01
- **Refines:** D-012. **Resolves:** O-010.
- **Decision:**
  - Model actions use a strict deterministic structured output schema.
  - If a response hits the completion cap but parses as a complete valid action or answer under that schema, process it normally.
  - If the response is truncated and cannot be parsed as a complete valid action or answer, terminate as an incomplete / budget-truncated response.
  - No LLM judge and no human interpretation.
- **Specs updated:** `METRICS_SPEC.md`, `ARCHITECTURE.md`, `POLICY_SPEC.md`, `TEST_PLAN.md`.

## D-022 — Calibrated-VoI operational definition
- **Date:** 2026-10-01
- **Refines:** D-003, D-013, D-017. **Resolves:** O-006 (remaining sub-points recorded as O-011 to O-015).
- **Decision:**
  - Planner side-channel output:
    ```json
    {
      "p_success_now": "[0,1]",
      "candidate_actions": [
        {
          "action": "...",
          "consumer": "schema|exploration|clarification|commit",
          "p_success_after": "[0,1]"
        }
      ]
    }
    ```
  - ONE shared isotonic calibrator maps the planner probability estimate to the calibrated probability of eventual task success.
  - For candidate action a: `Gain(a) = max(0, CalibratedP_after(a) − CalibratedP_now)`; `VoI(a) = Gain(a) / estimated_incremental_tokens(a)`.
  - Select the highest-VoI permitted action, including a deterministic commit / SQL option.
  - No other learned value model, regressor, or cost predictor.
  - Estimated incremental action cost uses a deterministic / empirical lookup derived from validation / pilot data, not a learned predictive model. Both estimated and actual incremental token cost are logged.
  - Deterministic tie-breaking: (1) higher VoI; (2) lower estimated token cost; (3) fixed action-priority order.
  - Calibration labels: the binary eventual task success (`execution_success`) of each validation episode, from the deterministic execution grader against benchmark gold results, on a validation split disjoint from the frozen 100-task main set. Calibration is not implemented if that would require inventing missing benchmark labels.
- **Specs updated:** `POLICY_SPEC.md`, `METRICS_SPEC.md`, `TEST_PLAN.md`.

## D-023 — Planner prompt invariant wording
- **Date:** 2026-10-01
- **Supersedes in part:** the "same planner prompt" wording in D-013 and D-017.
- **Decision:** The invariant is the same planner prompt template / instruction contract and the same structured output schema under all four policies. The runtime model-visible state may legitimately differ, because the policies expose different schemas, exploration results, and clarification history.
- **Specs updated:** `POLICY_SPEC.md`, `ARCHITECTURE.md`, `TEST_PLAN.md`.

## D-024 — Side channel is never model-visible under any policy
- **Date:** 2026-10-01
- **Supersedes in part:** D-017 (which excluded only Static-All, Retrieve-then-Load, and JIT / Progressive). **Resolves:** O-017.
- **Decision:** The structured VoI side channel must NOT be injected into model-visible state under any policy, including Calibrated-VoI. The planner side channel is controller metadata, consumed by the policy controller and logged in telemetry only; it is never inserted into model-visible prompts under any policy.
- **Specs updated:** `POLICY_SPEC.md`, `ARCHITECTURE.md`, `METRICS_SPEC.md`, `TEST_PLAN.md`.

## D-025 — VoI candidates are the three consumers; threshold stop rule
- **Date:** 2026-10-01
- **Supersedes in part:** D-022 (the `commit` candidate and "including a deterministic commit / SQL option"). **Resolves:** O-012, O-013.
- **Decision:**
  - `commit` is removed as a candidate action / consumer.
  - The three candidate consumers are `schema`, `exploration`, `clarification`.
  - The VoI policy selects the highest-VoI permitted consumer action.
  - If the maximum permitted consumer VoI is below the pilot-locked threshold, the policy stops acquiring information and proceeds to the fixed SQL path.
  - This is the defined role of `voi_thresholds`.
- **Specs updated:** `POLICY_SPEC.md`, `TEST_PLAN.md`.

## D-026 — Malformed non-truncated output
- **Date:** 2026-10-01
- **Resolves:** O-016.
- **Decision:**
  - Malformed structured output that is not truncated is retried if retry budget remains.
  - The model call is charged normally.
  - The retry itself is not a logical turn.
  - After the retry cap is exhausted, the episode terminates with the canonical termination reason `malformed_output`.
  - No LLM judge or human interpretation.
- **Specs updated:** `METRICS_SPEC.md`, `ARCHITECTURE.md`, `TEST_PLAN.md`.

## D-027 — Calibration protocol remains open
- **Date:** 2026-10-01
- **Qualifies:** D-022 (its calibration-label statement is not an approved fitting protocol). **Does not resolve:** O-014.
- **Decision:**
  - The real calibration protocol remains explicitly OPEN, because `p_success_after` is a counterfactual estimate and its empirical calibration labels have not been defined.
  - The repository may define the mathematical VoI interface and mock tests.
  - The repository must NOT implement a real isotonic-fitting procedure until the validation-label protocol is approved.
- **Specs updated:** `POLICY_SPEC.md`, `TEST_PLAN.md`, `README.md`.

## D-028 — Calibrated-VoI deterministic tie-breaking
- **Date:** 2026-10-01
- **Refines:** D-022, D-025. **Resolves:** O-011.
- **Decision:** For Calibrated-VoI candidate actions, ties are broken in this order:
  1. higher VoI;
  2. lower estimated incremental token cost;
  3. consumer priority: `schema` > `exploration` > `clarification`;
  4. within the same consumer, lexicographic canonical action identifier.
- Selection is purely deterministic and must not depend on random ordering.
- **Specs updated:** `POLICY_SPEC.md`, `TEST_PLAN.md`.

## D-029 — Definition of incremental token cost
- **Date:** 2026-10-01
- **Refines:** D-022. **Resolves:** O-015.
- **Decision:**
  - Estimated incremental token cost is the expected number of NEW model INPUT tokens introduced into the next model-visible state as a direct consequence of taking candidate action a. Examples: schema expansion = newly exposed schema tokens; exploration = returned environment / tool-result tokens that enter the next prompt; clarification = simulator / user-response tokens that enter the next prompt.
  - Excluded: the model-call tokens that selected the action (already incurred); future SQL-generation completion tokens; future verifier completion tokens; retriever compute.
  - The lookup may be keyed by consumer + canonical action / tool identifier + tool space where appropriate.
  - Both `estimated_incremental_tokens` and `actual_incremental_tokens` are logged.
  - No fitted predictive cost model is permitted.
- **Specs updated:** `POLICY_SPEC.md`, `METRICS_SPEC.md`, `TEST_PLAN.md`.

## D-030 — No permitted consumer action
- **Date:** 2026-10-01
- **Refines:** D-025. **Resolves:** O-018.
- **Decision:** If no permitted consumer action exists, the Calibrated-VoI policy returns `PROCEED_TO_SQL` and the fixed SQL path continues normally. The episode is not terminated solely because no consumer action is available.
- **Specs updated:** `POLICY_SPEC.md`, `METRICS_SPEC.md`, `TEST_PLAN.md`.

## D-031 — Calibration wording and permitted implementation scope
- **Date:** 2026-10-01
- **Qualifies:** the calibration wording in D-003, D-013, and D-022 ("calibrated with the planned isotonic regression", "ONE shared isotonic calibrator maps …"); these describe a reserved component, not a fixed research calibration method. **Does not resolve:** O-014.
- **Decision:**
  - A single shared isotonic calibrator is reserved. Its real fitting / validation protocol remains OPEN.
  - The repository may implement only: the side-channel schema; `Cal(...)`; `Gain(...)`; `VoI(...)`; the threshold rule; deterministic selection; mock-calibrator tests.
  - The real calibration artifact remains blocked until the validation-label protocol is explicitly approved.
- **Specs updated:** `POLICY_SPEC.md`, `TEST_PLAN.md`.

---

## Open items

Resolved:

- **O-001** — resolved by D-011.
- **O-002** — resolved by D-012.
- **O-003** — partially resolved by D-013; remainder is O-006.
- **O-004** — resolved by D-014.
- **O-005** — resolved by D-015.
- **O-006** — resolved by D-022 (sub-points not fixed by D-022 are O-011 to O-015).
- **O-007** — resolved by D-018.
- **O-008** — resolved by D-019.
- **O-009** — resolved by D-020.
- **O-010** — resolved by D-021.
- **O-012** — resolved by D-025 (`commit` candidate removed).
- **O-013** — resolved by D-025 (`voi_thresholds` is the stop threshold).
- **O-016** — resolved by D-026.
- **O-017** — resolved by D-024.
- **O-011** — resolved by D-028.
- **O-015** — resolved by D-029.
- **O-018** — resolved by D-030.

Still open (require a human decision; not to be resolved during implementation):

- **O-014 (from D-022, D-027): calibration validation-label protocol.** `p_success_after(a)` is a counterfactual estimate, and no empirical label for it has been defined. To be decided: which side-channel estimates enter the fit of the single isotonic calibrator; what observed outcome labels each of them; under which policy the validation episodes are generated (Calibrated-VoI itself needs the calibrator); and how multiple decision points from one episode are weighted. Until approved, no real isotonic fitting is implemented.
