# Architecture Specification v1.1

Revision v1.1 incorporates research decisions D-001, D-004, D-006, D-010, D-011, D-012, D-013, D-015, D-017, D-020, D-021, D-023, D-024, D-026 from `DECISIONS.md`.

## 1. High-level architecture

```text
USER QUERY
    |
    v
FIXED ORCHESTRATOR
Planner / SQL Writer / Verifier
    |
    v
ACTIVE ALLOCATION POLICY
    |
    +-------------------+-------------------+-------------------+
    |                   |                   |
    v                   v                   v
(a) SCHEMA         (b) EXPLORE        (c) ASK USER
    |                   |                   |
    +-------------------+-------------------+
                        |
                        v
                    MCP LAYER
             +----------+----------+
             |          |          |
             v          v          v
          DATABASE      KB     USER SIMULATOR
             |
             v
           SQLite
             |
             v
       SQL RESULT / ERROR
             |
             v
        VERIFICATION
             |
             v
       FINAL SUBMISSION
             |
             v
   DETERMINISTIC EXECUTION GRADE
```

## 2. Fixed orchestrator
The fixed agent consists of:

- Planner: interprets the task, tracks ambiguity/current state, and follows the active policy interface. Under all four policies, the same planner call, the same planner prompt template / instruction contract, and the same structured output schema generate a structured uncertainty / action-value signal (D-023). The signal is controller metadata: it is consumed by the policy controller, logged in telemetry, and never inserted into model-visible prompts under any policy, including Calibrated-VoI (D-024). Only Calibrated-VoI uses it for allocation. There is no separate confidence-elicitation call (D-013, `POLICY_SPEC.md`). The runtime model-visible state may legitimately differ across policies, because the policies expose different schemas, exploration results, and clarification history.
- SQL Writer: synthesizes read-only SQL from available information.
- Verifier: executes/inspects the result and enables bounded self-correction.

Execution is read-only. Static SQL validation must block write/destructive statements.

Model actions use a strict deterministic structured output schema; validity of a response is decided only by parsing against that schema (D-021). A malformed non-truncated response is retried within the retry cap; once the cap is exhausted the episode terminates with `malformed_output` (D-026).

Self-correction is bounded to at most three retries. Every retry is fully chargeable under the same accounting rules as any other model call.

## 3. MCP servers
Three servers are required:

### Database server
- `list_tables`
- `describe_table`
- `sample_values`
- `execute_sql`

### Knowledge-base server
- `search_knowledge_base`
- `get_column_description`

### User-simulator server
- `ask_user`

### Harness capability
- `submit_answer`

The MCP layer is common infrastructure. All four policies operate against the same MCP interfaces.

### Policy-layer control plane (not MCP tools)
Some policy operations act on what the agent can see rather than on the environment. These live in the policy layer, not in the MCP registry:

- JIT schema expansion (name + summary → full schema) is a policy-layer control-plane operation, not a ninth MCP tool. It belongs to the schema-exposure consumer (D-001).

Control-plane operations never change the agent-visible registry size (25 / 100 / 400) or the eight real MCP tools.

## 4. Policy gate placement
The allocation policy sits between the fixed orchestrator's current state and budget-consuming information-acquisition actions.

At an allocation point, the system has a current episode state containing at least:

- original user request;
- available tool information;
- known schema/environment facts;
- prior tool results;
- conversation/clarification history;
- prior SQL and execution results;
- remaining token budget;
- retry count;
- current logical turn.

The active policy determines whether and how a consumer action is granted under the locked policy definition.

### Budget enforcement at the model-call boundary (D-006)
Before every model call, the runner sets max completion tokens to `min(remaining_episode_budget, model_context_limit − current_input_tokens)` (D-020). No episode may exceed B, and input is never silently truncated. If no budget remains, the call is not issued and the episode terminates with `budget_exhausted`; if the input exceeds the context window, it terminates with `context_limit_exceeded`. Capped truncations are handled per D-012 / D-021. Budget currency and the exact procedure are defined in `METRICS_SPEC.md` §7a.

## 5. Important architectural rule
Do not interpret the architecture as:

```text
Static-All -> no exploration
Retrieve-then-Load -> no clarification
JIT -> no exploration
Calibrated-VoI -> only policy with MCP
```

That is incorrect.

Instead:

```text
All four policies -> SAME MCP environment

Policy difference -> how exposure / allocation decisions are made

Calibrated-VoI -> only policy that dynamically reallocates across
                  schema, exploration, and clarification in-episode
```

## 6. Logical turn vs. implementation event
A logical turn is a benchmark-level interaction cycle. Multiple implementation events may occur inside it, including LLM calls, MCP calls, tool results, and bookkeeping.

Do not automatically equate every low-level LLM call with a new logical conversational turn. Internal LLM calls, verifier calls, and internal retries do not automatically create new logical turns (D-004). Initialization-time exposure is not a logical turn, and every actual `execute_sql` interaction is one (D-018, D-019). The enumerated counting rules are in `METRICS_SPEC.md` §9 (D-011).

The precise logical-turn rule is defined in `METRICS_SPEC.md` and must be implemented consistently across all policies.

## 7. Runtime environments
Development path:

```text
Local machine -> tests / mocks / lightweight components
```

Experimental path:

```text
Colab GPU -> Qwen3-8B inference
           -> MCP servers
           -> SQLite / KB / user simulator
           -> experiment runner
           -> durable ledger/checkpoint files
```

All non-LLM components must be testable with a mock LLM backend so GPU inference is not required for routine development tests.

## 8. State and resumability
The runner must checkpoint after each completed episode and be able to resume after a Colab interruption without rerunning completed episode IDs.

Every result must be attributable to:

`experiment_version + query_id + policy + tool_space + seed + episode_id`

## 9. No explicit ReAct factor
The fixed agent may use structured internal reasoning, but explicit ReAct is not a separate architecture or experimental condition. Any prompting strategy is frozen across all four policies.

## 10. Observability / playground UI (non-experimental) (D-010)
A planned Streamlit UI is a non-experimental observability / playground layer.

- It must not alter the research engine or experimental semantics.
- It is not part of the experimental code path and is not a research variable or condition.
- The research engine (orchestrator, policies, MCP servers, accounting, grader, runner) must not depend on the UI.

Playground run mode (D-015):

- The playground may run live episodes through the same research engine.
- Playground controls may vary policy, tool space, token budget, task, and seed.
- Playground results are written to a separate playground-results location and never enter the controlled experiment ledger (`METRICS_SPEC.md` §13).
- Controlled experiment mode remains frozen and reproducible.
