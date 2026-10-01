# Policy Specification v1.1

Revision v1.1 incorporates research decisions D-001, D-002, D-003, D-007, D-008, D-009, D-013, D-014, D-017, D-021 through D-031 from `DECISIONS.md`.

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

### Accounting (D-002)
The upfront full database schema is charged to the schema-exposure consumer (consumer (a), bucket 1), not to exploration.

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

### Callability (D-008)
Retrieval happens exactly once. Tools outside the retrieved set are not callable for the remainder of the episode. An attempted call to a tool outside the retrieved set is a denied action (see Common rules, Denied actions).

### Retriever cost (D-009)
Retriever compute is not part of the model-token budget B. Retrieval latency is logged separately from model and tool latency.

## Policy 3 — JIT / Progressive

### Definition
Expose tool names and one-line summaries initially. Full tool schemas and deeper database/knowledge-base details are revealed only when requested/needed under the policy contract.

### Research role
Incremental/lazy exposure condition.

### Important behavior
JIT is not a requirement to use every information route. A route can remain unused when it is unnecessary.

### Schema expansion mechanism (D-001)
Expanding a tool from name + one-line summary to its full schema is a policy-layer control-plane operation. It is not a ninth MCP tool and does not appear in the agent-visible tool registry. Schema expansion belongs to the schema-exposure consumer (consumer (a), bucket 1).

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

### Calibration (D-031)
A single shared isotonic calibrator is reserved for mapping the planner's verbalised probability estimates to calibrated probabilities. Its real fitting / validation protocol remains OPEN (`DECISIONS.md` O-014); see "Calibration protocol — OPEN" below. Whatever protocol is approved, the calibrator and the decision threshold are frozen before the main experiment.

### Planner side-channel signal (D-003, D-013, D-017, D-023, D-024)
- The fixed planner generates the same structured uncertainty / action-value signal under all four policies. The same planner call, the same planner prompt template / instruction contract, and the same structured output schema are part of the fixed orchestrator under all four policies.
- The signal is controller metadata. It is consumed by the policy controller and logged in telemetry only. It is never inserted into model-visible prompts or model-visible state under any policy, including Calibrated-VoI.
- Only Calibrated-VoI consumes the signal, for allocation.
- Calibrated-VoI must not introduce a separate confidence-elicitation LLM call.
- The only fitted model permitted is the single shared isotonic calibrator reserved below. No additional learned value model, regressor, or cost predictor may be introduced.

### Planner side-channel schema (D-021, D-022)
The side channel uses the strict deterministic structured output schema (D-021):

```json
{
  "p_success_now": "<number in [0,1]>",
  "candidate_actions": [
    {
      "action": "<action identifier>",
      "consumer": "schema | exploration | clarification",
      "p_success_after": "<number in [0,1]>"
    }
  ]
}
```

Candidate actions belong to exactly the three budget consumers: `schema`, `exploration`, `clarification`. There is no `commit` candidate (D-025).

### Value computation (D-022, D-031)
`Cal` denotes the single shared isotonic calibrator reserved for mapping a planner probability estimate to a calibrated probability of eventual task success. The same `Cal` is applied to `p_success_now` and to every `p_success_after`. Its real fitting / validation protocol remains OPEN (O-014); until it is approved, `Cal` exists only as an interface exercised with mock calibrators.

For each candidate action `a`:

```text
Gain(a) = max(0, Cal(p_success_after(a)) − Cal(p_success_now))
VoI(a)  = Gain(a) / estimated_incremental_tokens(a)
```

Selection (D-025):

- Compute `VoI(a)` for every permitted candidate consumer action.
- If the maximum permitted consumer VoI is below the pilot-locked threshold (`voi_thresholds`, `PILOT_LOCK_REQUIRED`), the policy stops acquiring information and the episode proceeds to the fixed SQL path.
- Otherwise, the policy selects the highest-VoI permitted consumer action.
- If no permitted consumer action exists, the policy returns `PROCEED_TO_SQL` and the fixed SQL path continues normally. The episode is not terminated solely because no consumer action is available (D-030).

Deterministic tie-breaking among consumer actions, in order (D-028):

1. higher VoI;
2. lower estimated incremental token cost;
3. consumer priority: `schema` > `exploration` > `clarification`;
4. within the same consumer, lexicographic order of the canonical action identifier.

Selection is purely deterministic and must not depend on random ordering (including registry order or seed-shuffled tool position).

### Incremental cost estimate (D-022, D-029)
Definition: `estimated_incremental_tokens(a)` is the expected number of NEW model INPUT tokens introduced into the next model-visible state as a direct consequence of taking candidate action `a`.

- schema expansion: the newly exposed schema tokens;
- exploration: the returned environment / tool-result tokens that enter the next prompt;
- clarification: the simulator / user-response tokens that enter the next prompt.

Excluded:

- the model-call tokens that selected the action (already incurred);
- future SQL-generation completion tokens;
- future verifier completion tokens;
- retriever compute.

Estimation and logging:

- The estimate comes from a deterministic / empirical lookup derived from validation / pilot data. No fitted predictive cost model is permitted.
- The lookup may be keyed by consumer + canonical action / tool identifier + tool space, where appropriate.
- Lookup contents: `PILOT_LOCK_REQUIRED` (`voi_incremental_cost_lookup`).
- Both `estimated_incremental_tokens` and `actual_incremental_tokens` are logged for every Calibrated-VoI selection. `actual_incremental_tokens` is measured under the same definition: the new input tokens that the action's result actually introduced into the next model-visible state. It is measured from the labelled prompt segments introduced by the selected action, including deterministic wrapper text directly caused by that insertion. It must not be computed as an unrestricted difference between successive full prompts.

### Calibration protocol — OPEN (D-022, D-027; `DECISIONS.md` O-014)
The real calibration protocol is NOT defined.

`p_success_after(a)` is a counterfactual estimate (the probability of success had action `a` been taken). Its empirical calibration labels have not been defined, so the validation-label protocol for fitting the single shared isotonic calibrator is not approved.

Constraints that already hold:

- exactly one isotonic calibrator, shared across consumers and actions;
- validation data disjoint from the frozen 100-task main set;
- any label must come from the project's deterministic execution grader against benchmark gold results; no LLM judge, no human interpretation, no constructed labels;
- the fitted calibrator is frozen before the main experiment and versioned with the experiment.

Implementation constraint until the protocol is approved (D-027, D-031). The repository may implement only:

- the side-channel schema;
- `Cal(...)` as an interface;
- `Gain(...)`;
- `VoI(...)`;
- the threshold rule;
- deterministic selection (including tie-breaking and `PROCEED_TO_SQL`);
- mock-calibrator tests.

The repository must NOT implement a real isotonic-fitting procedure, and the real calibration artifact remains blocked until the validation-label protocol is explicitly approved.

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

"Same prompts" means the same prompt templates / instruction contracts and the same structured output schemas under all four policies (D-023). The runtime model-visible state may legitimately differ across policies, because the policies expose different schemas, exploration results, and clarification history.

### Denied actions (D-007, D-014)
When a policy denies an action:

- the model-facing message is identical for all policies:
  `ACTION_NOT_PERMITTED: The requested action is unavailable in the current state.`
- the detailed denial reason is telemetry only and never enters a model prompt;
- the denied action consumes zero action/tool cost;
- it does not create a logical turn;
- any resulting denial message that enters a later model prompt is charged normally as model input;
- the model call that proposed the denied action is still charged, as is every model call (see `METRICS_SPEC.md`).

## Policy isolation test
A policy switch must be configurable without changing the fixed-agent code path, MCP server definitions, database contents, or evaluation code.
