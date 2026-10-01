# Architecture Specification v1.0

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

- Planner: interprets the task, tracks ambiguity/current state, and follows the active policy interface.
- SQL Writer: synthesizes read-only SQL from available information.
- Verifier: executes/inspects the result and enables bounded self-correction.

Execution is read-only. Static SQL validation must block write/destructive statements.

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

Do not automatically equate every low-level LLM call with a new logical conversational turn.

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
