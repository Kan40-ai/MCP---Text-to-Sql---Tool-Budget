# UI Specification v1.0 — Playground and Observability Dashboard

## 0. Status and authority

- This document is the source of truth for the future Streamlit playground / observability dashboard.
- The UI is a non-experimental observability and playground layer (`ARCHITECTURE.md` §10, `DECISIONS.md` D-010, D-015).
- This document is subordinate to the research contracts: `PROJECT_SPEC.md`, `ARCHITECTURE.md`, `POLICY_SPEC.md`, `METRICS_SPEC.md`, `TEST_PLAN.md`, `EXPERIMENT_MATRIX.yaml`, `DECISIONS.md`. If anything here conflicts with them, the research contracts win and this document must be corrected.
- Nothing in this document defines or changes research semantics. Policy descriptions, metric names, termination reasons, and accounting quantities shown in the UI are taken verbatim from the research contracts.
- No UI code exists yet. Section 23 gives non-binding implementation guidance.

---

## 1. Purpose

The UI lets a user:

1. understand how the MCP Text-to-SQL agent works;
2. run a playground episode interactively;
3. select among the four policies;
4. change playground-only parameters;
5. observe budget allocation across the three research consumers;
6. see MCP tool activity;
7. inspect the agent trajectory;
8. inspect generated SQL and verification;
9. compare the four policies on the same task;
10. understand why Calibrated-VoI selected a particular information action, at the level of the controller's recorded decision.

The UI contains no research logic. The research engine never depends on the UI. The UI is an observability / control layer only.

---

## 2. Design language

A professional AI-research / systems-observability dashboard, not a generic chatbot.

Required characteristics:

- clean, modern, engineering-dashboard aesthetic;
- strong visual hierarchy;
- high information density without clutter;
- clear cards and panels; compact tables;
- progress bars, badges, expandable trace events;
- monospace areas for SQL, JSON, and tool payloads;
- minimal animation (state changes only);
- clear success / error / warning states, always with text and icon, never colour alone;
- visual emphasis on the three budget consumers, the central research variable.

Appropriate for a research demonstration, a graduate course presentation, an engineering architecture demo, and a benchmark / observability interface.

Avoid:

- chat bubbles as the primary layout;
- heavy gradients and decorative animation;
- large empty whitespace;
- unnecessary configuration controls;
- exposing research-locked internals as editable controls.

---

## 3. Top-level application structure

### 3.1 Persistent header

```text
MCP TEXT-to-SQL TOOL BUDGET LAB                                   [ MODE BADGE ]
Cost-aware orchestration of schema exposure, environment exploration, and user clarification.
```

The mode badge is always visible and is one of:

- `PLAYGROUND — NON-EXPERIMENTAL`
- `CONTROLLED EXPERIMENT — FROZEN CONFIGURATION`

### 3.2 Tabs

| Tab | Mode | Data source |
|---|---|---|
| 1. Playground | Playground | live playground episode; playground-results location |
| 2. Policy Comparison | Playground (live comparison) or Controlled Experiment (ledger view) — the active source is always labelled | playground-results location, or controlled experiment ledger / summaries (read-only) |
| 3. Experiment Dashboard | Controlled Experiment | controlled experiment ledger and backend-produced summaries (read-only) |

All three tabs share the same header, typography, status vocabulary, and consumer colour semantics.

---

## 4. Data boundary

The backend / research engine is authoritative. The UI visualizes backend telemetry.

The UI MUST NOT:

- compute research metrics independently (accuracy, AUC, Pareto counts, accuracy/$, retrieval diagnostics, bootstrap intervals, pass^k);
- implement or emulate policies;
- alter token accounting or attribute tokens to buckets itself;
- alter budget decisions;
- modify experiment configuration;
- alter grading;
- mutate the MCP registry;
- write to, rewrite, or delete from the controlled experiment ledger.

Data the UI consumes:

| Data | Producer | UI access |
|---|---|---|
| Per-step trace events (`METRICS_SPEC.md` §3) | research engine | read |
| Per-episode run record (`METRICS_SPEC.md` §2) | research engine | read |
| Cell / aggregate summaries, frontier, Pareto, AUC, retrieval diagnostics | backend analysis | read |
| Playground episodes | research engine, launched by the playground | launch + read; written only to the playground-results location |
| Controlled experiment ledger (`results/`) | experiment runner | read-only |

Every record displayed carries its `run_mode`. The UI never mixes `playground` and `experiment` records in one table, chart, or aggregate.

Simple presentational arithmetic on backend fields (e.g. `B − cumulative` for a progress bar, percentage of B for bar width, formatting) is permitted. Any derived research metric must come from the backend.

---

## 5. Playground view

Interactive single-episode environment. Always badged `PLAYGROUND — NON-EXPERIMENTAL`. Results go only to the playground-results location and never enter the controlled experiment ledger.

### 5.1 Controls (left panel)

Only the playground controls permitted by D-015 are exposed: policy, tool space, episode token budget, task, seed.

**Policy** — dropdown:

- Static-All
- Retrieve-then-Load
- JIT / Progressive
- Calibrated-VoI

Directly below, a one-line description taken from `POLICY_SPEC.md`:

| Policy | One-line description |
|---|---|
| Static-All | All visible tool schemas and the full database schema are exposed at initialization. |
| Retrieve-then-Load | The top-k candidate tools are retrieved once before execution; only retrieved tools are loaded and callable. |
| JIT / Progressive | Tool names and one-line summaries are exposed first; full schemas and deeper details are revealed only when requested. |
| Calibrated-VoI | Each next information action is chosen across schema, exploration, and clarification by value per incremental token; below threshold, the agent proceeds to SQL. |

**Tool Space** — dropdown: `25`, `100`, `400`.

**Episode Token Budget B** — numeric input (playground only). The default value is an implementation choice and must not be presented as the experimental B, which is `PILOT_LOCK_REQUIRED`.

**Task** — two modes:

- *Benchmark Task*: Task pool `[ ▼ ]`, Task ID `[ ▼ ]`. The pool offers only playground-eligible tasks (§5.4).
- *Custom Question*: natural-language text area (see 5.3 for limits).

**Seed** — integer input.

**Primary controls:** `[ ▶ RUN EPISODE ]`, `[ ▶▶ RUN ALL 4 POLICIES ]`, `[ ↻ RESET ]`.

- `RUN EPISODE` runs one playground episode with the selected policy.
- `RUN ALL 4 POLICIES` runs the four-policy playground comparison (§12.3) for the current task, tool space, budget, and seed, and opens the Policy Comparison tab on the result.

Not exposed as controls in any mode (shown read-only in the Frozen Configuration card where relevant): model, decoding parameters (temperature, top-p, etc.), retry cap, planner / orchestrator topology, verifier configuration, tokenizer, calibration artifact, SQL grader, MCP implementation, prompts.

### 5.2 Wireframe

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ MCP TEXT-to-SQL TOOL BUDGET LAB             [ PLAYGROUND — NON-EXPERIMENTAL ] │
│ Cost-aware orchestration of schema exposure, exploration, and clarification  │
├──────────────────────────────┬───────────────────────────────────────────────┤
│ PLAYGROUND CONTROLS          │ RUN STATUS                       ● RUNNING    │
│                              │ Policy  JIT / Progressive   Task  042         │
│ Policy                       │ Tool space  25              Seed  1           │
│ [ JIT / Progressive ▼ ]      │                                               │
│ Tool names and one-line      │ TOKEN BUDGET   7,420 / 10,000   (2,580 left)  │
│ summaries first; …           │ Schema exposure   ████████░░░░░░  2,340       │
│                              │ Exploration       █████░░░░░░░░░  1,420       │
│ Tool Space   [ 25 ▼ ]        │ Clarification     ██░░░░░░░░░░░░    410       │
│ Budget B     [ 10,000 ]      │ ─ fixed-agent ─   ███████░░░░░░░  3,050       │
│ Task  (•) Benchmark ( ) Custom│ ─ residual ─     ░░░░░░░░░░░░░░    200       │
│ Pool [ Demo / fixture ▼ ]    │                                               │
│ Task ID [ 042 ▼ ]            │ Turns 4 · Tool calls 3 · Clarifications 1     │
│ Seed    [ 1 ]                │ Latency 5.82 s                                │
│                              │                                               │
│ [ ▶ RUN EPISODE ]            │                                               │
│ [ ▶▶ RUN ALL 4 POLICIES ]    │                                               │
│ [ ↻ RESET ]                  │                                               │
├──────────────────────────────┴───────────────────────────────────────────────┤
│ LIVE AGENT TRACE (§7)                         │ MCP ACTIVITY (§8)             │
├───────────────────────────────────────────────┼───────────────────────────────┤
│ SQL / VERIFICATION (§10)                      │ CALIBRATED-VoI DECISION (§11) │
└───────────────────────────────────────────────┴───────────────────────────────┘
```

Numbers in wireframes are illustrative only.

The run-status header stays pinned while the trace scrolls.

### 5.3 Custom Question mode

A custom question has no benchmark gold result and no benchmark user-simulator specification. Therefore:

- execution accuracy is not computed; the final status is shown as `UNGRADED` (never PASS / FAIL);
- the target database must be chosen explicitly, from playground-eligible environment databases (§5.4);
- custom-question runs are playground-only and never appear in the Experiment Dashboard or in any controlled experiment ledger.

**Interactive clarification (U-3).** For custom questions, `ask_user` is answered by the human playground user, not by the controlled benchmark user simulator:

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ CLARIFICATION REQUESTED — TURN 4          [ HUMAN RESPONSE — NON-EXPERIMENTAL ]│
├──────────────────────────────────────────────────────────────────────────────┤
│ Agent asks:                                                                  │
│   "Should 'balance' mean the current balance or the original loan amount?"   │
│                                                                              │
│ Your response:                                                               │
│ ┌──────────────────────────────────────────────────────────────────────────┐ │
│ │                                                                          │ │
│ └──────────────────────────────────────────────────────────────────────────┘ │
│                                                     [ SUBMIT RESPONSE ]      │
└──────────────────────────────────────────────────────────────────────────────┘
```

- The episode pauses in state `WAITING FOR HUMAN RESPONSE` (§17) until the user submits a response. An empty response cannot be submitted.
- The submitted text is returned to the agent through the `ask_user` interaction; how it is counted is the backend's normal accounting (reply text is charged when it enters the next prompt). The UI does not count it.
- The controlled benchmark user simulator is never used for custom questions.
- Episodes with human responses are explicitly non-experimental. They are written only to the playground-results location and never enter the controlled experiment ledger.
- Latency for such episodes does not redefine the research latency metric. The UI may display three separately labelled values:
  - agent / system latency;
  - human response time (always separately identifiable);
  - total end-to-end elapsed time.
- Controlled-experiment latency is governed exclusively by the frozen backend `latency_measurement_convention` (`PILOT_LOCK_REQUIRED`) and is never changed by the UI.
- In `RUN ALL 4 POLICIES` with a custom question, each of the four episodes that asks for clarification shows its own clarification prompt, labelled with its policy; responses are never shared between episodes.

### 5.4 Playground task pool (U-1)

The frozen 100-task main evaluation set is never exposed in the Playground. It is accessible only to the controlled experiment runner.

Playground-eligible tasks:

| Pool | Content |
|---|---|
| Demo / synthetic | demonstration tasks authored for the playground |
| Fixture | test-fixture tasks and databases used by the test suite |
| Approved validation | non-main validation tasks explicitly approved for playground use |

Rules:

- The task picker lists only playground-eligible tasks. Main-set task IDs are not listed, searchable, or enterable.
- Eligibility comes from backend task metadata (the backend marks which tasks are playground-eligible). The UI does not decide eligibility, and it rejects any task the backend does not mark as eligible.
- Custom-question mode cannot be used to load a main-set task by ID or by pasting its text against its gold result; custom questions are always `UNGRADED`.
- The Experiment Dashboard may display controlled-runner results for main-set tasks read-only (§13); that is not free-form experimentation.

---

## 6. Run status panel and budget visualization

### 6.1 Always-visible fields

active policy · task · tool-space size · token budget B · tokens consumed · tokens remaining · logical turns · tool calls · clarification turns · latency · current execution state (§17).

### 6.2 Budget visualization

```text
TOKEN BUDGET (charged to B)                     Consumed 7,420 / 10,000
                                                Remaining 2,580

RESEARCH CONSUMERS
  Schema exposure     ██████████░░░░░░░░░░   2,340
  Exploration         ██████░░░░░░░░░░░░░░   1,420
  Clarification       ██░░░░░░░░░░░░░░░░░░     410

FIXED-AGENT STAGES
  Planning / SQL generation / execution results / self-correction / verification
                      ████████░░░░░░░░░░░░   3,050   [expand for per-bucket]
  Residual / system overhead
                      ░░░░░░░░░░░░░░░░░░░░     200

  Raw prompt 6,180 · Cached prompt 2,900 (logged separately) · Completion 1,240
```

Requirements:

- B, consumed, remaining, schema, exploration, clarification, fixed-agent, and residual tokens are always shown as exact numbers next to the bars.
- The three research consumers are grouped and visually emphasized as competing claims on one finite budget; fixed-agent stages are visually subordinate.
- The fixed-agent group expands to the five fixed-agent buckets (`METRICS_SPEC.md` §4, buckets 4–8).
- The residual / system-overhead bucket (`METRICS_SPEC.md` §5) is shown separately; it is never merged into a consumer.
- Zero usage of any consumer is shown as `0` with an empty bar and no warning styling. Using fewer than three consumers is a valid outcome.
- Cached prompt tokens are shown separately and are not subtracted from consumed budget (`METRICS_SPEC.md` §7a).
- All values come from backend telemetry (bucket attribution is a backend responsibility).

### 6.3 Two token quantities — must not be confused

| Quantity | Meaning | Where shown |
|---|---|---|
| Charged to B (by bucket) | Cumulative raw input + completion tokens across all model calls, attributed to buckets by labelled prompt segment. Content that stays in context is re-charged on every subsequent call. | Budget visualization (§6.2) |
| Incremental insertion (`actual_incremental_tokens`) | New input tokens that one action introduced into the next model-visible state, measured from that action's labelled segments (D-029). | Trace events (§7), VoI panel (§11) |

Every number in the UI that refers to tokens is labelled with which of these it is. The UI never sums incremental-insertion values to reconstruct budget consumption.

Both `estimated_incremental_tokens` and `actual_incremental_tokens` are backend-produced. The UI displays them and never computes, estimates, or corrects either value.

---

## 7. Live agent trace

A main visual component: the episode as an expandable chronological trace grouped by logical turn.

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ LIVE AGENT TRACE                                          [expand all] [▾]   │
├──────────────────────────────────────────────────────────────────────────────┤
│ INIT     │ SCHEMA EXPOSURE   initial name/summary exposure (25 tools)  0 turns│
│ TURN 1   │ SCHEMA EXPOSURE   expanded tool schema: describe_table  +438 new   │
│ TURN 2   │ EXPLORATION       list_tables()                         +212 new   │
│ TURN 3   │ EXPLORATION       describe_table("loans")               +516 new   │
│ TURN 4   │ CLARIFICATION     ask_user(…) → simulator reply         +184 new   │
│ —        │ DENIED            sample_values(…)   ACTION_NOT_PERMITTED  0 turns │
│ TURN 5   │ SQL / EXECUTION   execute_sql(…)                                    │
│ TURN 6   │ SUBMIT            submit_answer(…)                                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

Rules:

- Turn labels come from the backend's `logical_turn_id`. The UI does not count turns itself.
- Initialization-time exposure is shown as `INIT` and is not a logical turn (D-019).
- Denied actions are shown in place, marked `DENIED`, and are not logical turns (D-007).
- Internal LLM calls (planner / SQL writer / verifier) and internal retries are shown as nested sub-events inside the turn they belong to, not as turns (D-004, D-011, D-018).
- Every actual `execute_sql` interaction is its own turn, including during verification or retry (D-018).
- "+N new" is `actual_incremental_tokens` (§6.3).

Expanded event content (as available in telemetry):

event type · logical turn · step id · consumer / bucket · action / tool · input (arguments) · output (result summary, expandable to full payload in monospace) · estimated incremental tokens (Calibrated-VoI) · actual incremental tokens · model input / output / cached tokens for the step · cumulative model tokens · remaining budget · completion cap and `completion_cap_binding` · `finish_reason` · `output_parse_valid` · step latency · denied / not denied · relevant telemetry fields.

Chain-of-thought rule: the UI never displays hidden reasoning or thinking text. It shows action summaries, structured outputs, and token counts only. If a model call produced reasoning tokens, only their count is shown.

---

## 8. MCP activity panel

```text
MCP ACTIVITY
  DB SERVER              ✓ list_tables  ✓ describe_table  ○ sample_values  ✓ execute_sql
  KB SERVER              ○ search_knowledge_base  ○ get_column_description
  USER SIMULATOR         ✓ ask_user
  HARNESS                ✓ submit_answer
  POLICY CONTROL PLANE   ✓ schema expansion (JIT; not an MCP tool)
```

- Legend uses text + icon: `✓ called`, `○ not called`, `⊘ denied`, `✕ error`, with a call count per tool.
- The eight real MCP tools are always listed. Distractor tools are summarized as a count (e.g. "distractor calls: 0") and expandable.
- JIT schema expansion is shown under a separate "policy control plane" row, never as an MCP tool (D-001).
- Each interaction expands to: tool name · server · arguments · result summary (full payload in monospace) · tool input / output tokens (labelled transport diagnostic) · model input tokens when the result entered a prompt · latency · allowed / denied.
- Denied interactions are visually distinct (badge + icon + text), never colour alone.

---

## 9. Denied action display

```text
⊘ DENIED   sample_values(table="loans")

MODEL-FACING MESSAGE
  ACTION_NOT_PERMITTED: The requested action is unavailable in the current state.

TELEMETRY-ONLY REASON                                   [ technical ▸ ]
  <denial_reason from telemetry>

Cost: 0 action/tool tokens · Logical turn: none
```

- The model-facing message is shown verbatim (D-014).
- The telemetry-only reason is in a collapsed, explicitly labelled technical section, and is visually separated from the model-facing message.
- The panel states that the denial consumed zero action / tool cost and created no logical turn (D-007).

---

## 10. SQL / verification panel

Sections, in order:

1. **User question** — benchmark task text or custom question.
2. **Clarification history** — `ask_user` questions and simulator replies, if any.
3. **Generated SQL** — monospace, syntax-highlighted; one entry per attempt (attempts after self-correction or retry are numbered).
4. **SQL safety validation** — read-only check, statement-type check, result of the static validator (`ARCHITECTURE.md` §2).
5. **Execution** — success / error, error text in monospace, row count, result preview table.
6. **Verification** — verifier outcome per attempt; retry count vs retry cap.
7. **Final result** — submitted result preview.
8. **Final status** — one of:
   - `✓ PASS` / `✕ FAIL` — benchmark task graded by the backend's deterministic execution grader;
   - `UNGRADED` — custom question (§5.3);
   - the episode's termination state (§17) if no answer was submitted.

The UI never grades. PASS / FAIL is the backend's `execution_success`.

---

## 11. Calibrated-VoI decision panel

Shown only when the active policy is Calibrated-VoI. One panel per policy decision point, navigable by turn.

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ CALIBRATED-VoI DECISION — decision point before TURN 4    [ CALIBRATOR: MOCK ]│
├──────────────────────────────────────────────────────────────────────────────┤
│ p_success_now 0.41 → Cal 0.38                                                │
│                                                                              │
│ Candidate action        Consumer       p_after  Cal   Gain   Est.tok  VoI     │
│ expand: describe_table  schema         0.43     0.40  0.02   420     4.8e-5  │
│ describe_table("loans") exploration    0.49     0.47  0.09   600     1.5e-4  │
│ ask_user                clarification  0.52     0.50  0.12   300     4.0e-4  │
│ sample_values(…)        exploration    —        —     —      —       NOT PERMITTED│
│                                                                              │
│ Threshold 2.0e-4   Max permitted VoI 4.0e-4 ≥ threshold                      │
│ Tie-break: not needed                                                        │
│ SELECTED → CLARIFICATION: ask_user          actual incremental tokens: 184   │
└──────────────────────────────────────────────────────────────────────────────┘
```

Display:

- `p_success_now` and its calibrated value;
- per candidate: action identifier, consumer, `p_success_after`, calibrated value, Gain, `estimated_incremental_tokens`, VoI;
- non-permitted candidates listed as `NOT PERMITTED` (never selectable);
- threshold value and the comparison against the maximum permitted VoI;
- which tie-break rule decided the selection, if one was needed (D-028);
- selected action, or `PROCEED_TO_SQL` (below threshold, or no permitted consumer action — D-025, D-030);
- after the action: `actual_incremental_tokens` beside the estimate.

Rules:

- All values are read from the controller's logged decision (`planner_side_channel`, `voi_decision`, cost fields). The UI does not recompute VoI, Gain, calibrated values, or estimated / actual incremental tokens; all are backend-produced.
- The panel is labelled as a controller-decision record. It does not explain private model reasoning and adds no narrative.
- The panel notes that the side channel is controller metadata, never shown to the model (D-024).
- Calibrator status (U-4): Calibrated-VoI may run in Playground mode before O-014 is resolved, using only the mock calibrator and a placeholder threshold (D-027, D-031). Every such run is labelled exactly:

  `CALIBRATOR: MOCK — NOT A RESEARCH CALIBRATION`

  The label appears on the VoI decision panel, the run-status panel, the run's row in any table, its column in Policy Comparison, every chart point or series that includes it, and any exported playground record view.
- The threshold shown in the playground is labelled `PLACEHOLDER THRESHOLD` until `voi_thresholds` is pilot-locked.
- No real calibration artifact is loaded or offered in the Playground while O-014 is open.

### 11.1 Other policy-specific panels (compact)

- **Static-All**: initialization exposure summary — tool schemas exposed, DB schema exposed, tokens at initialization.
- **Retrieve-then-Load**: retrieved set (k tools, ranked, with retrieval scores if logged), retrieval latency (logged separately, D-009), gold-tool hit indicators when available from backend diagnostics, out-of-set denial count (D-008).
- **JIT / Progressive**: exposure state — tools at summary level vs expanded, with expansion turn numbers.

---

## 12. Policy Comparison view

Purpose: inspect the same task under the four policies and compare measured run-level outcomes.

### 12.1 Constant-conditions banner

```text
POLICY COMPARISON                                     SOURCE: [ PLAYGROUND RUNS ]
Same task · Same tool space · Same B · Same seed · Same model configuration
Task 042 · Tool space 25 · B 10,000 · Seed 1
```

- If any compared run differs in task, tool space, B, seed, or model configuration, the comparison is not displayed; the UI shows which field differs.
- The data source (playground runs or controlled experiment ledger) is always shown and never mixed.
- Playground comparisons use only playground-eligible tasks (§5.4).

### 12.2 Table

Single-task, single-seed comparison:

```text
┌────────────────────┬────────────┬────────────────────┬─────────────┬────────────────┐
│                    │ Static-All │ Retrieve-then-Load │ JIT / Prog. │ Calibrated-VoI │
├────────────────────┼────────────┼────────────────────┼─────────────┼────────────────┤
│ Execution success  │ ✓ PASS     │ ✓ PASS             │ ✕ FAIL      │ ✓ PASS         │
│ Termination        │ submitted  │ submitted          │ submitted   │ submitted      │
│ Tokens (raw)       │ 18,230     │ 11,420             │ 9,120       │ 8,310          │
│ Derived USD        │ …          │ …                  │ …           │ …              │
│ Latency            │ …          │ …                  │ …           │ …              │
│ Logical turns      │ …          │ …                  │ …           │ …              │
│ Tool calls         │ 6          │ 5                  │ 5           │ 4              │
│ Clarifications     │ 0          │ 1                  │ 1           │ 2              │
│ Schema / Expl. / Clar. tokens │ … │ …                │ …           │ …              │
└────────────────────┴────────────┴────────────────────┴─────────────┴────────────────┘
```

- For a single run, execution success is shown as PASS / FAIL, never as a percentage.
- Percentages (accuracy) appear only for aggregates provided by the backend, with the number of episodes (n) shown.
- Columns are always in the fixed order above. No column is highlighted, ranked, or labelled "best", "winner", or "recommended". Measured differences only.
- Failed and terminated runs (budget, context limit, malformed output, turn cap, runtime failure) are shown, never hidden.

### 12.3 Live four-policy comparison in the Playground (U-2)

`[ ▶▶ RUN ALL 4 POLICIES ]` launches four independent playground episodes, one per policy, with:

- the same task, tool space, token budget B, seed, and model configuration;
- a freshly reset initial state for each episode.

Independence rules:

- The four episodes share no episode state: no conversation / clarification history, exposed schemas, exploration results, retrieval results, cached prompt state, budget counters, turn counters, or retry counters carry over from one to another.
- Each episode has its own episode id and its own trace; each is drillable on its own (§14).
- Execution order and any concurrency are implementation choices; results must not depend on them.
- Results are playground-only: written to the playground-results location, badged `PLAYGROUND — NON-EXPERIMENTAL`, and never entered into the controlled experiment ledger.
- Progress is shown per policy (four status chips using the §17 states). If one episode fails or terminates early, the other three still complete, and the failed column shows its terminal state.
- A Calibrated-VoI column run with the mock calibrator carries `CALIBRATOR: MOCK — NOT A RESEARCH CALIBRATION` (§11).

### 12.4 Charts

A. accuracy vs token cost · B. accuracy vs derived USD · C. token allocation by consumer (stacked, three consumers + fixed-agent + residual) · D. latency by policy · E. tool calls and clarification turns · F. accuracy–cost frontier.

Every chart states: task subset · tool-space size · B · seed aggregation · single run vs aggregate · number of episodes per point · data source (playground / experiment).

Rules:

- Single-run charts carry a `SINGLE RUN — NOT AN AGGREGATE` label and show no error bars, confidence intervals, or significance markers.
- Confidence intervals, frontiers, AUC, and Pareto markers are drawn only from backend-produced values.
- Derived USD is labelled with its pricing convention and "derived, not a vendor invoice" (`METRICS_SPEC.md` §7).
- Exact numeric values are available next to every chart (table or tooltip + table).

---

## 13. Experiment Dashboard

Read-only view of controlled experiment results. Badged `CONTROLLED EXPERIMENT — FROZEN CONFIGURATION`.

Filters: Tool space `[All | 25 | 100 | 400]` · Policy `[All | …]` · Seed `[All | 1 | 2 | 3]` · Task subset `[All | …]`. Filter values come from `EXPERIMENT_MATRIX.yaml`.

Shows (all backend-produced):

execution accuracy · total raw tokens · completion tokens · cached tokens · derived USD · latency · logical turns · tool calls · clarification turns · retry count · termination-reason breakdown · accuracy / $ · AUC accuracy–cost · Pareto-dominance information · retrieval diagnostics (gold-tool recall@k, precision, MRR) · seed reliability (bootstrap CIs, pass^k where applicable).

Also shows: experiment version, number of completed / expected episodes (e.g. of 3,600), and checkpoint / resume status as reported by the runner.

The dashboard never launches, edits, re-runs, or deletes controlled experiment episodes (see §16).

---

## 14. Run-level vs trace-level

The UI preserves the two-level telemetry model (`METRICS_SPEC.md` §2–3).

**Run level** — one row / card per episode: episode id · policy · task · tool space · seed · execution success · termination reason · tokens · derived USD · latency · turns · tool calls · clarifications · run mode.

**Trace level** — many events per episode: turn · step · consumer / bucket · action · incremental tokens · cumulative tokens · remaining budget · latency · denial · finish reason · etc.

Clicking a run row drills down into its trace (§7 layout, read-only for experiment runs). The two levels are visually distinct (summary table vs event timeline) and labelled `RUN-LEVEL METRICS` and `TURN-LEVEL TRACE`.

---

## 15. Playground vs controlled experiment

| | Playground | Controlled experiment |
|---|---|---|
| Badge | `PLAYGROUND — NON-EXPERIMENTAL` | `CONTROLLED EXPERIMENT — FROZEN CONFIGURATION` |
| Policy, tool space, task, seed | user-selectable | filter / view only, limited to matrix values |
| Budget B | user-selectable | locked (pilot-locked value) |
| Launch episodes | yes | no (see §16) |
| Writes | playground-results location only | none |
| Reads | playground results | controlled experiment ledger and summaries |

Records from one mode never appear in the other mode's tables, charts, or aggregates.

---

## 16. Controlled Experiment mode and Frozen Configuration card

Controlled experiment episodes are produced only by the experiment runner (`ARCHITECTURE.md` §7–8). The UI is not part of the experimental code path (`ARCHITECTURE.md` §10), so Controlled Experiment mode in the UI is read-only: it views the ledger, it does not launch experiment runs.

Frozen Configuration card (values read from the experiment configuration, never editable):

```text
FROZEN CONFIGURATION — experiment v1.0
  Model                    Qwen3-8B                 LOCKED
  Precision / quantization <value | PILOT_LOCK_REQUIRED>
  Serving backend          <as recorded in experiment configuration>
  Decoding parameters      <value | not yet specified>
  Orchestrator             Planner / SQL Writer / Verifier   LOCKED
  Retry cap                3                        LOCKED
  Episode budget B         <value | PILOT_LOCK_REQUIRED>
  Max logical turns        <value | PILOT_LOCK_REQUIRED>
  Retriever and k          <value | PILOT_LOCK_REQUIRED>
  VoI threshold            <value | PILOT_LOCK_REQUIRED>
  VoI cost lookup          <value | PILOT_LOCK_REQUIRED>
  Calibration artifact     BLOCKED — O-014 open
  Grading                  execution-based, no LLM judge     LOCKED
  Policies                 4 · Tool spaces 25 / 100 / 400 · Seeds 1 / 2 / 3
```

- Unresolved pilot-lock values are displayed as `PILOT_LOCK_REQUIRED`, never as a guessed or default number.
- The card is also available read-only in the Playground (collapsed), so users see which settings the playground does not let them change.

---

## 17. Run states

Each state has a text label and an icon; colour is supplementary.

**Transient states** (episode in progress):

| State | Meaning |
|---|---|
| IDLE | no episode configured |
| READY | configured, not started |
| RUNNING | episode in progress |
| WAITING FOR TOOL | MCP call in flight |
| WAITING FOR USER SIMULATOR | `ask_user` in flight (benchmark simulator) |
| WAITING FOR HUMAN RESPONSE | `ask_user` in a custom-question playground episode, awaiting the user's submitted response (§5.3) |
| GENERATING SQL | SQL-writer model call in flight |
| VERIFYING | verifier step in flight |

**Event state** (not terminal):

| State | Meaning |
|---|---|
| DENIED ACTION | an action was denied; the episode continues |

**Terminal states** (mapped one-to-one from the backend's `termination_reason`; the UI never infers them):

| State | Source |
|---|---|
| COMPLETED — PASS | answer submitted, `execution_success = true` |
| COMPLETED — FAIL | answer submitted, wrong answer |
| COMPLETED — UNGRADED | answer submitted, custom question |
| TURN CAP REACHED | maximum logical-turn cap |
| BUDGET EXHAUSTED | `budget_exhausted` |
| CONTEXT LIMIT EXCEEDED | `context_limit_exceeded` |
| MALFORMED OUTPUT | `malformed_output` |
| FAILED — RUNTIME ERROR | unrecoverable runtime failure (shown distinctly from a wrong answer) |

---

## 18. Visual semantics

One consistent mapping across all views:

| Category | Used for |
|---|---|
| Schema exposure | consumer (a) bars, trace events, chips |
| Exploration | consumer (b) |
| Clarification | consumer (c) |
| SQL / execution | SQL generation and `execute_sql` |
| Verification | verifier steps |
| Fixed-agent (neutral) | other fixed-agent buckets, residual |
| Denied | denied actions |
| Error | runtime errors, terminal failures |
| Success | PASS, successful steps |

- The palette is chosen at implementation time; it must be restrained, accessible (sufficient contrast, colour-blind safe), and consistent in light and dark themes.
- Every category also has a text label and an icon / glyph. Colour is never the only carrier of meaning.

---

## 19. Interaction design

The user can: launch an episode · reset · expand / collapse events · expand all · inspect SQL · inspect MCP activity · inspect token accounting · inspect the final result · compare policies · filter experiment results · drill down from a run to its trace · copy SQL / JSON payloads.

- The run-status header remains visible while long traces scroll.
- Launching (`RUN EPISODE` or `RUN ALL 4 POLICIES`) is disabled while an episode or comparison is running; reset is confirmed if one is in progress.
- Submitting a human clarification response (§5.3) is the only input accepted while an episode is paused.
- Locked controls are visibly locked (lock icon + tooltip naming the controlling spec).

---

## 20. Responsiveness

Desktop is the primary target; the design must remain usable on smaller laptop screens.

- collapsible control side panel;
- tabbed views;
- vertical scrolling;
- responsive card widths;
- compact tables with horizontal scroll inside the table only, never the page.

Mobile is not a primary target.

---

## 21. Accessibility and clarity

- readable font sizes; monospace for code and payloads;
- never colour alone;
- explicit labels on every number (unit and which token quantity, §6.3);
- locked controls distinguished from editable ones;
- playground vs experiment always distinguished;
- technical terms explained via short tooltips (e.g. "logical turn", "consumer", "VoI");
- exact numeric values alongside every chart.

---

## 22. Research-integrity rules

The UI must never:

- change research semantics;
- imply a policy is superior through language, ordering, colour, or highlighting;
- rank or score policies beyond backend-produced project metrics;
- hide failed or terminated runs;
- silently omit budget, context-limit, malformed-output, or turn-cap terminations;
- merge playground and experiment data;
- write to, rewrite, or delete from the experiment ledger;
- introduce a UI-only policy behaviour;
- show model chain-of-thought;
- present mock-calibrator Calibrated-VoI runs as research results;
- expose the frozen 100-task main evaluation set to free-form Playground experimentation;
- route human playground responses into the controlled benchmark simulator or the experiment ledger;
- let Playground comparison episodes share state;
- present placeholder values as pilot-locked values.

---

## 23. Implementation guidance (non-binding)

The eventual implementation may use Streamlit. Keep UI code modular and outside the research engine package. Suggested structure:

```text
ui/
  app.py
  components/
    controls.py  status.py  budget.py  trace.py  mcp_panel.py
    sql_panel.py  voi_panel.py  comparison.py  experiment_dashboard.py
  views/
    playground.py  policy_comparison.py  experiment_dashboard.py
```

- The UI imports the research engine's public interfaces and telemetry readers; the research engine never imports the UI (`TEST_PLAN.md` §7a: engine tests pass with the UI package absent).
- On the local Mac the playground runs live episodes with MockLLM; real-model playground runs require the experimental backend environment.
- This is guidance only; no files are created by this specification.

---

## 24. User journey

```text
USER
 ↓ select mode (Playground / view Controlled Experiment)
 ↓ configure allowed controls
 ↓ run (Playground only)
 ↓ observe budget consumption
 ↓ watch allocation trace
 ↓ watch MCP interactions
 ↓ inspect SQL / verification
 ↓ see final task result
 ↓ inspect cost / tokens / latency
 ↓ compare policies
 ↓ inspect aggregate experiment results
```

---

## 25. Traceability

| UI requirement | Research contract |
|---|---|
| non-experimental UI, separate playground results | D-010, D-015, `ARCHITECTURE.md` §10, `METRICS_SPEC.md` §13 |
| three consumers vs fixed-agent buckets, residual | `PROJECT_SPEC.md` §4, `METRICS_SPEC.md` §4–5 |
| budget currency, cached tokens | D-005, `METRICS_SPEC.md` §7a |
| logical-turn display | D-004, D-011, D-018, D-019 |
| denied-action display | D-007, D-014 |
| JIT expansion as control plane | D-001 |
| VoI panel fields, threshold, tie-break, PROCEED_TO_SQL | D-022, D-024, D-025, D-028, D-029, D-030 |
| mock-calibrator labelling | D-027, D-031, O-014; UI decision U-4 |
| playground task pool excludes main set | `AGENTS.md` Research integrity; UI decision U-1 |
| independent four-policy playground runs | `POLICY_SPEC.md` Policy isolation test; UI decision U-2 |
| human clarification for custom questions | D-015; UI decision U-3 |
| terminal states | `METRICS_SPEC.md` §10, D-020, D-021, D-026 |

---

## 26. Resolved UI decisions

- **U-1 — Playground task pool.** Resolved: the frozen 100-task main evaluation set is not exposed for free-form Playground experimentation. The Playground may use synthetic / demo tasks, fixture tasks, and approved non-main validation tasks. The main set is controlled-runner only (§5.4).
- **U-2 — Policy Comparison source.** Resolved: live four-policy comparison is kept in the Playground via `RUN ALL 4 POLICIES`, launching four independent episodes with the same task, tool space, B, seed, and model configuration and a reset initial state, sharing no episode state; results remain playground-only (§12.3). Ledger-based comparison remains available read-only.
- **U-3 — Custom questions and clarification.** Resolved: for custom questions, `ask_user` is answered through an interactive human response control. This is explicitly non-experimental, never uses the controlled benchmark simulator, and never enters the controlled experiment ledger (§5.3).
- **U-4 — Calibrated-VoI before O-014.** Resolved: Calibrated-VoI may run in Playground mode before O-014 is resolved, using only the mock calibrator and a placeholder threshold. Every such run is labelled `CALIBRATOR: MOCK — NOT A RESEARCH CALIBRATION` (§11).
