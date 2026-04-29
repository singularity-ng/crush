# singularity-crush — Specification

**Version:** 0.1.0-draft  
**Status:** Research / Pre-implementation  
**Authors:** singularity-ng

---

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## Table of Contents

1. [Overview](#1-overview)
2. [Definitions](#2-definitions)
3. [Data Model](#3-data-model)
4. [Phase State Machine](#4-phase-state-machine)
5. [Orchestration Loop](#5-orchestration-loop)
6. [Worker Attempt Lifecycle](#6-worker-attempt-lifecycle)
7. [Prompt Contract](#7-prompt-contract)
8. [Context Budget](#8-context-budget)
9. [Supervision](#9-supervision)
10. [Hook Pipeline](#10-hook-pipeline)
11. [Workspace Management](#11-workspace-management)
12. [Worktree Isolation](#12-worktree-isolation)
13. [Verification Gates](#13-verification-gates)
14. [Configuration](#14-configuration)
15. [Model Routing](#15-model-routing)
16. [Knowledge Layer](#16-knowledge-layer)
17. [Persistent Agents](#17-persistent-agents)
18. [Inter-Agent Messaging](#18-inter-agent-messaging)
19. [Observability](#19-observability)
20. [Failure Taxonomy](#20-failure-taxonomy)
21. [Trust Boundary](#21-trust-boundary)
22. [Distributed Execution](#22-distributed-execution)
23. [Plugin Extension Points](#23-plugin-extension-points)
24. [Secret Management](#24-secret-management)
25. [CLI Commands](#25-cli-commands)
26. [Conformance Checklist](#26-conformance-checklist)

---

## 1. Overview

singularity-crush is an autopilot layer built on top of [charmbracelet/crush](https://github.com/charmbracelet/crush). Crush is an interactive coding agent — a human drives it turn by turn. singularity-crush adds a harness that drives Crush autonomously through a structured phase sequence (research → plan → execute → verify → complete) without human intervention per unit, while the human watches or steers.

singularity-crush is a fork of Crush. It MUST NOT rebuild what Crush already provides:

- Agent loop via `charm.land/fantasy`
- Multi-provider LLM (Anthropic, OpenAI, Gemini, Groq, Bedrock, Azure, Ollama) via `charm.land/catwalk`
- MCP client (`modelcontextprotocol/go-sdk`)
- LSP integration
- SQLite state via `ncruces/go-sqlite3`
- Bubbletea TUI + Lipgloss
- Tool execution (bash, file read/write, grep, web search, sourcegraph)
- Agent Skills open standard (`internal/skills/`)
- Permission service with pubsub, persistent grants, hook pre-approval
- PreToolUse hook system with allow/deny/halt, input rewriting, multi-hook aggregation

This specification covers only what singularity-crush adds. Behaviour already specified by Crush is inherited.

**Project-level conformance.** singularity-crush itself MUST enforce godoc on every exported identifier in its harness packages via a CI check (`scripts/specs-check.go` — an AST walk, no external linter dependency). This applies to singularity-crush's own development; it is not a runtime gate against user projects.

---

## 2. Definitions

**Unit** — the atomic unit of work. Has a type (`milestone`, `slice`, `task`), a phase, and an attempt counter. Units are ephemeral — they complete or fail and are archived.

**Phase** — a named stage of a unit's lifecycle. The harness owns all phase transitions; no other layer may transition a phase directly.

**Attempt** — one dispatch of a worker for a unit. A unit may accumulate multiple attempts across failures and retries.

**Turn** — one model call within an attempt. An attempt consists of one or more turns. The first turn receives the full task prompt; subsequent turns receive continuation guidance only.

**Session** — a top-level container with a stable ID, persisting across process restarts. The session holds the running state for all units, the context budget, and the supervisor state.

**Harness** — the layer between the agent loop (fantasy) and SF's orchestration logic (milestones, phases, git, worktrees). It owns: context budget, phase transitions, unit lifecycle hooks, session contract, observability, and supervision. Nothing in the planning or git layers MUST reach past the harness boundary into fantasy directly.

**Worker** — the process (local or SSH-remote) that executes one attempt. Spawned by the orchestrator.

**Orchestrator** — central process. Owns the scheduling loop, in-memory state, and all SQLite writes. Always runs locally even in distributed deployments.

**Hindsight** — the durable knowledge store backed by a running Hindsight service. Holds memories, learnings, and anti-patterns across sessions and projects. Not SQLite — a separate service reachable on the tailnet.

**Skill** — a `SKILL.md` file providing prompt guidance to the agent. Inspirational, not enforced.

**Workflow template** — a TOML file specifying the exact phase sequence the harness enforces for a class of work. Programmatic, not a suggestion to the agent.

**Tracker** — the external system from which work is pulled (Linear, GitHub Issues, Jira, or built-in SQLite). The tracker is authoritative for unit status; the local `units` row mirrors it. See § 3.3.

**Claim** — a soft lock recorded on a `units` row indicating the orchestrator is currently dispatching it. Stored as `claim_holder` (worker host or PID) and `claim_until` (UNIX ms expiry). A claim is released on terminal phase, worker exit, or claim expiry. Prevents two orchestrators or two workers picking up the same unit simultaneously.

---

## 3. Data Model

The orchestrator uses a single SQLite database (`~/.sf/sf.db`) for **orchestration state only**: sessions, units, phase transitions, blockers, gate results, benchmarks, circuit breakers, and persistent agents. **Knowledge** (memories, learnings, anti-patterns, codebase context) lives in Hindsight (§ 16), not SQLite.

The schema MUST be managed via sqlc and MUST use WAL mode:

```sql
PRAGMA journal_mode=WAL;
PRAGMA synchronous=NORMAL;
```

### 3.1 Core tables

```sql
CREATE TABLE sessions (
    id           TEXT PRIMARY KEY,
    status       TEXT NOT NULL,          -- idle | running | paused | interrupted | complete | failed
    created_at   INTEGER NOT NULL,
    updated_at   INTEGER NOT NULL
);

CREATE TABLE units (
    id           TEXT PRIMARY KEY,
    session_id   TEXT NOT NULL REFERENCES sessions(id),
    parent_id    TEXT REFERENCES units(id),  -- milestone has NULL parent; slice's parent is milestone; task's parent is slice
    type         TEXT NOT NULL,          -- milestone | slice | task
    phase        TEXT NOT NULL,
    phase_status TEXT NOT NULL,          -- pending | running | succeeded | failed | canceled | interrupted
    attempt      INTEGER NOT NULL DEFAULT 1,  -- 1 = first try, 2 = first retry, ...
    claim_holder TEXT,                   -- worker host/PID currently dispatching this unit; NULL = unclaimed
    claim_until  INTEGER,                -- UNIX ms; claim auto-expires at this time
    priority     INTEGER,                -- 1 (urgent) .. 4 (low); NULL sorts last
    tracker_kind TEXT,                   -- linear | github | jira | sqlite; matches tracker registry
    tracker_id   TEXT,                   -- external ID in the tracker (issue ID, ticket key, etc.)
    title        TEXT NOT NULL,
    description  TEXT,
    worker_host  TEXT,                   -- NULL = local; binary = SSH host
    workspace    TEXT,
    created_at   INTEGER NOT NULL,
    updated_at   INTEGER NOT NULL
);

CREATE TABLE phase_transitions (
    id           TEXT PRIMARY KEY,
    unit_id      TEXT NOT NULL REFERENCES units(id),
    from_phase   TEXT NOT NULL,
    to_phase     TEXT NOT NULL,
    reason       TEXT,
    transitioned_at INTEGER NOT NULL
);

CREATE TABLE task_blockers (
    task_id      TEXT NOT NULL,
    blocked_by   TEXT NOT NULL,          -- unit_id of upstream dependency
    PRIMARY KEY (task_id, blocked_by)
);

CREATE TABLE gate_results (
    id           TEXT PRIMARY KEY,
    unit_id      TEXT NOT NULL REFERENCES units(id),
    gate_name    TEXT NOT NULL,
    passed       INTEGER NOT NULL,
    attempt      INTEGER NOT NULL,
    max_retries  INTEGER NOT NULL,
    output       TEXT,                   -- truncated at 8KB
    duration_ms  INTEGER NOT NULL,
    recorded_at  INTEGER NOT NULL
);

CREATE TABLE session_blockers (
    id           TEXT PRIMARY KEY,
    session_id   TEXT NOT NULL REFERENCES sessions(id),
    event        TEXT NOT NULL,          -- GateBlocked | MergeConflict | Paused
    unit_id      TEXT,
    detail       TEXT,
    created_at   INTEGER NOT NULL,
    resolved_at  INTEGER
);

CREATE TABLE benchmark_results (
    id           TEXT PRIMARY KEY,
    model        TEXT NOT NULL,
    tier         TEXT NOT NULL,
    fingerprint  TEXT NOT NULL,          -- phase+complexity+project hash
    quality      REAL NOT NULL,          -- 0.0 .. 1.0
    latency_p50  INTEGER NOT NULL,       -- milliseconds
    cost_per_1k  REAL NOT NULL,
    sample_count INTEGER NOT NULL DEFAULT 1,
    recorded_at  INTEGER NOT NULL
);

CREATE TABLE circuit_breakers (
    model        TEXT PRIMARY KEY,
    tier         TEXT NOT NULL,
    tripped_at   INTEGER NOT NULL,
    resets_at    INTEGER NOT NULL,        -- UNIX ms; auto-reset deadline
    fail_count   INTEGER NOT NULL DEFAULT 3,
    reason       TEXT
);

CREATE TABLE schema_migrations (
    version      INTEGER PRIMARY KEY,
    applied_at   INTEGER NOT NULL,
    description  TEXT
);
```

### 3.2 Persistent agent tables

```sql
CREATE TABLE agents (
    id           TEXT PRIMARY KEY,
    name         TEXT NOT NULL UNIQUE,
    system       TEXT NOT NULL,          -- system prompt template
    model        TEXT NOT NULL,
    state        TEXT NOT NULL DEFAULT 'idle', -- idle | running | waiting | stopped
    created_at   INTEGER NOT NULL,
    last_active  INTEGER
);

CREATE TABLE agent_memory_blocks (
    agent_id     TEXT NOT NULL REFERENCES agents(id),
    label        TEXT NOT NULL,
    value        TEXT NOT NULL DEFAULT '',
    char_limit   INTEGER NOT NULL DEFAULT 2000,
    read_only    INTEGER NOT NULL DEFAULT 0,
    updated_at   INTEGER NOT NULL,
    PRIMARY KEY (agent_id, label)
);

CREATE TABLE agent_messages (
    id           TEXT PRIMARY KEY,
    agent_id     TEXT NOT NULL REFERENCES agents(id),
    seq          INTEGER NOT NULL,       -- monotonically increasing per agent
    role         TEXT NOT NULL,          -- user | assistant | tool_call | tool_return | system
    content      TEXT NOT NULL,
    tool_name    TEXT,
    created_at   INTEGER NOT NULL
);

CREATE TABLE agent_inbox (
    id           TEXT PRIMARY KEY,
    agent_id     TEXT NOT NULL REFERENCES agents(id),
    from_agent   TEXT NOT NULL,
    content      TEXT NOT NULL,
    delivered    INTEGER NOT NULL DEFAULT 0,
    created_at   INTEGER NOT NULL
);
```

`agent_inbox` is append-only. Rows MUST NOT be deleted or modified after insert. `delivered` is the only mutable field.

### 3.3 Task Tracker Integration

A **task tracker** is the external system from which the orchestrator pulls eligible work. The `units` row is a local mirror of an entry in some upstream tracker; the tracker is authoritative for issue state.

#### 3.3.1 Tracker interface

Every tracker plugin MUST implement:

```go
type Tracker interface {
    // Identifies which tracker_kind values this implementation handles.
    Kind() string

    // Fetch eligible candidate units. Called every poll tick.
    // The result is filtered locally before dispatch.
    FetchCandidates(ctx context.Context, opts FetchOpts) ([]TrackerUnit, error)

    // Re-fetch a single unit's authoritative state. Called between turns
    // and during reconciliation.
    FetchUnitState(ctx context.Context, trackerID string) (TrackerState, error)

    // Optional: write the unit's state back (e.g. mark in-progress).
    UpdateUnitState(ctx context.Context, trackerID string, state TrackerState) error
}

type TrackerUnit struct {
    TrackerID    string            // external identifier
    Title        string
    Description  string
    Priority     *int              // 1..4 or nil
    Labels       []string
    BlockedBy    []string          // upstream tracker IDs
    Metadata     map[string]string // tracker-specific fields
}

type TrackerState struct {
    Status   TrackerStatus // see § 3.3.2
    UpdatedAt time.Time
}
```

#### 3.3.2 Lifecycle states

The tracker MUST report each unit's status as one of:

| Status | Meaning | Eligible for dispatch? |
|---|---|---|
| `active` | Open, ready, in progress, queued | YES |
| `blocked` | Waiting on upstream | NO (re-evaluated next tick) |
| `done` | Completed, merged, closed-as-resolved | NO (terminal) |
| `cancelled` | Closed-as-wontfix, deleted, archived | NO (terminal) |
| `unknown` | Tracker returned a status the plugin couldn't classify | NO; logged as warning, treated as blocked |

**Non-active** = anything except `active`. A unit becoming non-active mid-run triggers `ReconciliationCancel` (§ 9.2) and the attempt enters `AttemptCanceled`.

#### 3.3.3 Built-in trackers

| `tracker_kind` | Source |
|---|---|
| `linear` | Linear GraphQL API |
| `github` | GitHub Issues + Projects |
| `jira` | Jira REST v3 |
| `sqlite` | Built-in local tracker — units defined in `.sf/tracker.toml`, stored in SQLite directly. Default for projects without an external tracker. |

Additional trackers MAY be added via the `Tracker` plugin interface (§ 23).

#### 3.3.4 ID mapping

`units.tracker_kind` + `units.tracker_id` together identify the upstream entity. The harness MUST treat `(tracker_kind, tracker_id)` as a unique key — re-fetching the same upstream entity reuses the same `units.id`.

#### 3.3.5 Failure handling

Tracker errors do NOT crash the orchestrator (§ 20). Specifically:

- `FetchCandidates` failure: skip this poll tick, retry next tick.
- `FetchUnitState` failure mid-attempt: log warning, continue the turn loop. The next poll tick will re-fetch.
- Persistent tracker outage: dispatched workers continue running on already-fetched state; no new dispatches occur until tracker recovers.

---

## 4. Phase State Machine

### 4.1 Phase enum

```go
type Phase int

const (
    PhaseResearch Phase = iota // map the problem, gather context
    PhasePlan                  // decompose into slices and tasks, get sign-off
    PhaseExecute               // write the code
    PhaseTDD                   // write tests for what was just built; red → green
    PhaseVerify                // run full test suite + lint + type check; gates pass
    PhaseReview                // structured self-review: correctness, style, security
    PhaseMerge                 // commit, push, open PR
    PhaseComplete              // unit done; result recorded; artifact archived
    PhaseReassess              // re-enter planning with failure context
    PhaseUAT                   // human acceptance; only when uat_dispatch = true
)
```

### 4.2 Standard flow

`Research → Plan → Execute → TDD → Verify → Review → Merge → Complete`

Permitted non-standard transitions:

| Trigger | Transition |
|---|---|
| Gate failure in Verify (attempt < max_retries) | `Verify → Execute` |
| Gate failure in Verify (attempt = max_retries) | `Verify → Reassess` |
| Review finds a real problem | `Review → Execute` |
| Merge conflict | `Merge → Reassess` |
| External cancellation | Any → (AttemptCanceled, no phase write) |

All other transitions are REJECTED at the harness boundary with a typed error. The harness MUST NOT silently allow invalid transitions.

### 4.3 Attempt state

Within each phase, individual dispatch attempts move through finer-grained states:

```go
type AttemptState int

const (
    AttemptPreparingWorkspace AttemptState = iota
    AttemptBuildingPrompt
    AttemptLaunchingAgent
    AttemptInitializingSession
    AttemptStreamingTurn
    AttemptFinishing
    AttemptSucceeded
    AttemptFailed
    AttemptTimedOut
    AttemptStalled           // stall_timeout exceeded since last agent event
    AttemptCanceled          // issue became non-active mid-run (reconciliation)
)
```

`AttemptCanceled` is distinct from `AttemptFailed`. It means the work was valid but the task was externally invalidated (deleted, moved to a terminal state, superseded). The harness MUST NOT retry a canceled attempt — it releases the slot and moves on.

### 4.4 Turn kind

```go
type TurnKind int

const (
    TurnFirst        TurnKind = iota // full rendered task prompt
    TurnContinuation                 // short continuation guidance, same thread
)
```

Turn 1 of every attempt is always `TurnFirst`. Turns 2+ are `TurnContinuation`. The harness determines `TurnKind`; the agent never does.

### 4.5 Workflow templates

A workflow template MUST be a TOML file in `.sf/workflows/<name>.toml`. The harness reads the template, constructs the phase sequence from it, and enforces it programmatically. The agent has no say in phase ordering or skipping.

```toml
# .sf/workflows/feature.toml
name            = "feature"
phases          = ["research", "plan", "execute", "tdd", "verify", "review", "merge", "complete"]
require_tdd     = true    # PhaseTDD is enforced; skipping is a gate violation
require_review  = true
max_retries     = 3       # per gate in PhaseVerify
max_reassess    = 2

# .sf/workflows/spike.toml
name            = "spike"
phases          = ["research", "plan", "execute", "complete"]
require_tdd     = false
require_review  = false
max_retries     = 0
```

The harness MUST fail startup if a configured workflow template references an unknown phase.

### 4.6 Phase transitions — implementation rules

1. All phase transitions MUST go through a single `Harness.Transition(ctx, from, to, reason)` method.
2. `Transition` MUST persist the `PhaseTransition` record to SQLite BEFORE the new phase begins. A crash mid-phase means on resume the harness re-enters the last committed phase cleanly (see § 4.7).
3. `Transition` MUST emit a pubsub `PhaseChange` event after the SQLite write. The TUI subscribes — it MUST NOT poll phase state directly.
4. The harness MUST set `Think: true` on the model config for `Research` and `Plan` phases. The agent does not control this.

### 4.7 Crash recovery

In-memory scheduler state is intentionally not persisted (§ 20.2). On restart, the orchestrator MUST follow this exact sequence:

1. **Acquire process lock** at `~/.sf/run.lock` (PID file). Stale lock (PID not in `/proc`) is cleaned and logged.
2. **Mark interrupted units.** All units with `phase_status = 'running'` are updated to `phase_status = 'interrupted'`. This is the only schema-level recovery action.
3. **Run startup cleanup** (§ 5.6) — move stale active artifacts to archive.
4. **Resume from the last committed phase boundary.** Each `interrupted` unit is treated as eligible for fresh dispatch; the worker re-enters at `unit.phase` with a new attempt number (`unit.attempt + 1`). The agent receives a `last_error` of `"resumed_after_crash"` so the prompt can warn the agent.
5. **Begin polling.** Tracker reconciliation runs first to catch any status changes that happened during the outage.

The harness MUST NOT replay tool calls. It MUST NOT attempt to "resume" a partial agent session. The crash recovery model is **fresh dispatch from the last persisted phase boundary**, not transparent continuation.

**Side effects are not rolled back.** A crash mid-Merge may have produced a partial commit, push, or PR. The agent on retry sees the existing commits and either continues from there or surfaces a conflict. This MUST be documented in the Merge phase prompt: "if you see existing commits from a previous attempt, integrate them; do not start over."

**Workspace state is preserved.** A crashed worker's workspace remains on disk; the next attempt reuses it (`ensure_workspace` returns `created=false`). The `before_run` hook is responsible for any cleanup (e.g. `git stash`, `npm clean`) appropriate for the project.

---

## 5. Orchestration Loop

### 5.1 Poll cycle

The orchestrator runs a single goroutine that polls on a configurable interval (default 1s). Each tick:

1. Re-check config stamp (§ 14.3).
2. Fetch eligible units from SQLite.
3. Apply priority sort (§ 5.2).
4. For each eligible unit (up to capacity), dispatch a worker.
5. Check running workers for stalled/timed-out attempts.
6. Write orchestrator snapshot to HTTP API state (§ 19.4).

The orchestrator MUST be the single authority for all in-memory scheduler state. No other goroutine writes scheduler state.

### 5.2 Priority ordering

When multiple units are eligible, the orchestrator sorts them:

1. **Explicit priority** — `priority` 1 (urgent) before 4 (low); `NULL` sorts last.
2. **Blocker-free first** — units with no non-terminal upstream blockers before blocked units.
3. **Phase order** — earlier phases first (Research before Execute) within the same priority bucket.
4. **Created-at** — oldest first as tie-breaker.
5. **Unit ID lexicographic** — final deterministic tie-breaker.

This ordering is re-evaluated fresh on every poll tick.

### 5.3 Blocker-aware dispatch

A unit MUST NOT be dispatched if any of its upstream dependencies (in `task_blockers`) are in a non-terminal state.

**Terminal** means `PhaseComplete`, `PhaseReassess` (resolved), or explicitly cancelled. **Non-terminal** means any other state, including `PhaseVerify` in progress.

A dependency that failed and was marked abandoned is terminal and MUST NOT block downstream dispatch.

Blocked units stay queued and are re-evaluated on the next poll tick. No backoff, no retry counter increment for a blocked wait.

### 5.4 Per-phase concurrency

The harness MUST NOT exceed `max_agents_by_phase[phase]` concurrent units in any given phase. When a phase slot is full, further dispatches for that phase wait until the next tick.

```toml
[harness.concurrency]
max_agents                   = 10
max_agents_by_phase.execute  = 4
max_agents_by_phase.tdd      = 4
max_agents_by_phase.verify   = 10
```

### 5.5 Continuation retry and exponential backoff

**After a normal (clean) exit** from a worker, the orchestrator MUST schedule a 1-second continuation retry to re-poll eligibility. If the unit is still active, a new session starts. If terminal, the claim is released. This is not a failure retry.

**After an abnormal exit**, exponential backoff. `attempt` is 1-indexed (first try = 1, first retry = 2, …):

```
delay = min(10s × 2^(attempt - 1), max_retry_backoff)
```

| Attempt | Delay before next dispatch |
|---|---|
| 1 (first try) | (no retry yet) |
| 2 (first retry) | 20 s |
| 3 | 40 s |
| 4 | 80 s |
| 5 | 160 s |
| 6+ | capped at `max_retry_backoff` (default 5 min) |

Configurable: `[harness] max_retry_backoff = "5m"`, `[harness] max_attempts = 6`.

### 5.6 Startup cleanup

On startup, the orchestrator MUST:

1. Scan `.sf/active/` for unit artifacts whose tasks are in terminal states.
2. Move stale active artifacts to `.sf/archive/` atomically (rename, not copy+delete).
3. Mark any running/claimed units as interrupted in SQLite.
4. Release all worker slots.

---

## 6. Worker Attempt Lifecycle

The exact sequence inside a single worker attempt:

```
run_worker_attempt(unit, attempt):
  # 1. Workspace
  workspace = create_or_reuse_workspace(unit.id, unit.worker_host)
  if workspace failed:
    fail_attempt(ErrWorkspaceCreation)

  # 2. Before-run hook (fatal)
  result = run_hook("before_run", workspace, unit)
  if result failed:
    fail_attempt(ErrHookFailed)

  # 3. Session start
  session = agent.start_session(cwd=workspace, model=route(unit.phase))
  if session failed:
    run_hook_best_effort("after_run", workspace, unit)
    fail_attempt(ErrAgentStartup)

  # 4. Turn loop
  turn = 1
  loop:
    kind = TurnFirst if turn == 1 else TurnContinuation
    prompt = build_prompt(unit, attempt, turn, kind)
    if prompt failed:
      agent.stop_session(session)
      run_hook_best_effort("after_run", workspace, unit)
      fail_attempt(ErrPromptRender)

    result = agent.run_turn(session, prompt)
    if result failed:
      agent.stop_session(session)
      run_hook_best_effort("after_run", workspace, unit)
      fail_attempt(result.error)

    # Re-check unit state between turns
    current_state = tracker.fetch_unit_state(unit.id)
    if current_state is non-active:
      break  # → AttemptCanceled

    if turn >= max_turns_per_attempt:
      break

    turn++

  # 5. Teardown
  agent.stop_session(session)
  run_hook_best_effort("after_run", workspace, unit)
  exit_normal()
```

Rules:
- `before_run` hook failure is fatal — the harness MUST fail the attempt without starting the session.
- `after_run` hook is always attempted, even after failure. Its failure is logged but MUST NOT change the attempt outcome.
- The unit state re-check between turns MUST happen before building the next turn prompt. A canceled unit MUST NOT receive another turn.

---

## 7. Prompt Contract

### 7.1 Template variables

Every prompt template MUST be rendered with a strict variable checker. An unknown variable in the template MUST cause `loadPrompt` to panic at startup rather than silently render an empty string.

Canonical variables for execute-task templates:

| Variable | Type | Notes |
|---|---|---|
| `unit_id` | string | Stable unit identifier |
| `unit_type` | string | `"milestone"` \| `"slice"` \| `"task"` |
| `phase` | string | Current phase name (`"execute"`, `"tdd"`, etc.) |
| `attempt` | int \| null | `null` on first dispatch; integer ≥ 1 on retry |
| `session_id` | string | Stable session UUID |
| `issue` | object | Full issue/task struct as flat map |
| `last_error` | string \| null | Injected automatically when `attempt >= 1` |

When adding a new `{{variable}}` to any template: (1) pass it in every `loadPrompt` call site, (2) add a placeholder in every test that renders that template, (3) recompile. Skipping either step causes a startup panic.

### 7.2 Continuation turns

A `TurnContinuation` MUST receive a short guidance prompt, not the full task prompt. The full prompt is already in the thread history — resending it inflates context and degrades model reasoning. The continuation prompt MUST NOT re-state the task description; it provides only steering context for the current turn.

### 7.3 Attempt variable semantics

The `attempt` variable enables prompt templates to give different instructions to retrying agents vs. fresh starts. A retry prompt SHOULD include: `"your previous attempt failed with: {{last_error}} — focus on that specifically."` The harness injects `last_error` automatically on `attempt >= 1`.

### 7.4 `turn_input_required` in auto-mode

When the agent raises `turn_input_required` during auto-mode, the harness MUST respond according to the `turn_input_required` config (default: `"soft"`):

- **`"soft"`** — inject `"This is a non-interactive session. Operator input is unavailable."` as a `user` role turn and let the session continue. The agent adapts.
- **`"hard"`** — end the attempt immediately, record `ErrTurnInputRequired`, schedule failure retry.

In interactive/step mode, the harness MUST surface the request to the user via the TUI and MUST NOT auto-respond. It waits up to `unit_timeout` before failing.

The harness MUST NOT leave a run stalled indefinitely waiting for interactive input in any mode.

---

## 8. Context Budget

### 8.1 Budget type

```go
type Budget struct {
    MaxTokens   int
    UsedTokens  int
    CompactAt   float64 // fraction e.g. 0.80
    HardLimitAt float64 // fraction e.g. 0.95
}

func (b *Budget) ShouldCompact() bool {
    return float64(b.UsedTokens)/float64(b.MaxTokens) >= b.CompactAt
}

func (b *Budget) AtHardLimit() bool {
    return float64(b.UsedTokens)/float64(b.MaxTokens) >= b.HardLimitAt
}
```

### 8.2 Rules

- The harness MUST update `UsedTokens` after every model response. The agent loop MUST NOT manage budget.
- When `ShouldCompact()` is true, the harness MUST trigger compaction before the next turn, not mid-turn.
- When `AtHardLimit()`, the harness MUST halt the current unit, snapshot state, and surface `ErrBudgetExhausted`. It MUST NOT let the agent proceed and hit a provider context error.
- Budget state MUST be persisted to SQLite after every turn so crash recovery can restore it.

### 8.3 Compaction

When compaction fires (budget at compact threshold):

1. Write a `session_summary` entry to Hindsight via `retain`.
2. Clear the hot cache (in-memory last-N turns).
3. Start the next turn with a fresh context window seeded by a `recall` from Hindsight.

Compaction MUST NOT truncate the window — it MUST replace it with a fresh recall. A truncated window loses structure; a recalled window gains relevance.

### 8.4 Token accounting precision

Provider responses arrive as either absolute thread totals or per-turn deltas. The harness MUST prefer absolute totals (`thread/tokenUsage/updated`-style events) and MUST track the last-reported total to compute deltas, preventing double-counting.

Aggregate totals (input, output, cache-read, cache-write, cost-usd) MUST accumulate in orchestrator state and be included in every runtime snapshot.

---

## 9. Supervision

### 9.1 Supervisor interface

The harness MUST run a supervisor goroutine alongside the agent loop. The supervisor communicates exclusively via pubsub — it MUST NOT touch agent state directly.

```go
type SupervisorCheck interface {
    Name() string
    Check(ctx context.Context, state SupervisorState) SupervisorSignal
}

type SupervisorSignal int

const (
    SignalOK    SupervisorSignal = iota
    SignalWarn                   // log, surface in TUI
    SignalPause                  // pause auto-loop, wait for user
    SignalAbort                  // stop unit, mark interrupted
)
```

### 9.2 Built-in checks

| Check | Trigger | Signal |
|---|---|---|
| `StuckLoop` | Same phase for > N turns with no successful tool calls | `SignalPause` |
| `BudgetWarning` | Context approaching compaction threshold | `SignalWarn` |
| `TimeoutCheck` | Unit running longer than `unit_timeout` | `SignalAbort` |
| `AbandonDetect` | Agent producing output with no tool calls | `SignalPause` |
| `GitDivergence` | Working branch diverged from base unexpectedly | `SignalPause` |
| `ReconciliationCancel` | Unit's underlying issue transitioned to non-active mid-run | `SignalAbort` (reason: `canceled_by_reconciliation`) |
| `BlockerCheck` | Upstream dependency moved to non-terminal state mid-run | `SignalPause` |
| `ModelUnavailable` | Provider returns "model not supported / not found" class error | `SignalAbort` immediately (not after timeout) |
| `CircuitBreaker` | Same model fails 3 consecutive times within a session | Trip circuit; `SignalAbort` on next dispatch to tripped model |

### 9.3 Circuit breaker

When the circuit trips for a model:

- Write circuit state to SQLite (`circuit_breakers` table — `model`, `tripped_at`, `resets_at`).
- Subsequent dispatches in that tier MUST skip the tripped model.
- Circuit auto-resets after 24 hours or on explicit `/sf reset-circuits`.
- The circuit state MUST survive a process restart.

### 9.4 Supervisor constraints

- The supervisor MUST NOT call `os.Exit` or panic.
- The supervisor MUST NOT write to agent state or SQLite unit state directly.
- The auto-loop acts on `SignalPause` and `SignalAbort`. The TUI shows warnings on `SignalWarn`.

---

## 10. Hook Pipeline

### 10.1 Events

The harness extends Crush's existing `internal/hooks/` with SF-specific events:

```go
const (
    // Existing Crush event
    EventPreToolUse    = "PreToolUse"

    // Unit lifecycle
    EventPreDispatch   = "PreDispatch"   // before a unit is dispatched; can block
    EventPostUnit      = "PostUnit"      // after a unit completes
    EventPhaseChange   = "PhaseChange"   // on phase transition

    // Auto-loop
    EventAutoLoop      = "AutoLoop"      // each iteration of the auto-loop

    // Worktree
    EventWorktreeCreate = "WorktreeCreate"
    EventWorktreeDelete = "WorktreeDelete"
    EventMergeReady     = "MergeReady"
    EventMergeConflict  = "MergeConflict"

    // Agent fleet
    EventAgentWake     = "AgentWake"     // target agent should start/resume
    EventAgentMessage  = "AgentMessage"  // message routed (TUI + tracing)
    EventAgentIdle     = "AgentIdle"     // agent completed its turn, inbox empty
)
```

### 10.2 UnitResult payload

PostUnit hooks receive:

```go
type UnitResult struct {
    UnitID        string
    UnitType      string        // "milestone" | "slice" | "task"
    Phase         Phase
    Verdict       string        // "success" | "failure" | "abandoned"
    Duration      time.Duration
    InputTokens   int
    OutputTokens  int
    CacheHits     int
    CostUSD       float64
    Model         string
    WorkerHost    string
    Error         error
    Learnings     []string
}
```

The payload is serialized to JSON and passed to hook subprocesses via stdin.

### 10.3 Hook execution rules

- PostUnit hooks run **sequentially**, not concurrently. The next dispatch MUST NOT begin until all PostUnit hooks have returned.
- A hook subprocess that exits non-zero for `PreDispatch` or `PostUnit` MUST trigger `SignalAbort`. The harness stops the session and marks it `SessionFailed`.
- A hook that times out (default 60s, configurable via `[harness.hooks] timeout_ms`) is killed and logged. A `PostUnit` hook timeout MUST NOT block the next dispatch.
- The git service subscribes to PostUnit via a hook and handles commits, branch creation, and push. The harness MUST NOT call `git` directly.
- Hindsight feedback (retain learnings, mark anti-patterns) is emitted from a built-in PostUnit hook (not a subprocess) — it calls the Hindsight client directly.
- PostUnit hook results MUST be written to the trace as child spans of the unit span.

### 10.4 Tool response contract

Every tool call — successful or not — MUST return a response in this shape:

```go
type ToolResponse struct {
    Success      bool          `json:"success"`
    Output       string        `json:"output"`
    ContentItems []ContentItem `json:"contentItems"`
}

type ContentItem struct {
    Type string `json:"type"` // always "inputText" for text results
    Text string `json:"text"`
}
```

For successful calls: `success = true`, `output` = result summary. For unsupported or failed calls: `success = false`, `output` = human-readable error, `contentItems` lists which tools are available in the current context. The shape MUST be consistent — the agent relies on `success` to distinguish real failures from tool-not-found errors.

If the agent calls a tool that is not registered, the harness MUST return a structured failure response and continue the session. It MUST NOT stall, panic, or exit on an unknown tool name.

### 10.5 Doc sync (sub-step of PhaseMerge)

Doc sync runs as the final sub-step of `PhaseMerge`, before transitioning to `PhaseComplete`. It is not a separate phase and not a post-merge hook; it is part of the Merge phase's contract.

The doc-sync sub-step:

1. Dispatches a `fast`-tier turn against the merged diff with a short prompt asking whether project-level docs (`ARCHITECTURE.md`, `CONVENTIONS.md`, `STACK.md`) need updating.
2. The agent emits a diff (possibly empty) to stdout.
3. If the diff is non-empty, the harness surfaces it to the TUI for user approval. On approval, it is committed as `docs: sync after {unit_id}` on the same branch and the merge hook is re-triggered.
4. On empty diff, the sub-step is a no-op and PhaseMerge proceeds to PhaseComplete.

Configuration:
- `[harness] doc_sync = false` disables the sub-step entirely.
- `[harness] doc_sync_auto_approve = true` skips the user prompt and commits the diff directly. Off by default.

---

## 11. Workspace Management

### 11.1 Naming

Workspace directories are derived from the unit identifier. The identifier MUST be sanitized: replace any character not in `[a-zA-Z0-9._-]` with `_`. This prevents path injection via issue identifiers containing slashes, `..`, or null bytes.

### 11.2 Symlink-aware path containment

Workspace path validation MUST use segment-by-segment canonicalization, not `filepath.EvalSymlinks` or `path.Clean` alone. A naive call can be defeated by a symlink that resolves outside the workspace root.

Algorithm:

```
resolveCanonical(path):
  segments = split(path)
  resolved = root
  for segment in segments:
    candidate = join(resolved, segment)
    stat = lstat(candidate)
    if stat == symlink:
      target = readlink(candidate)
      # expand target relative to current resolved prefix
      # restart segment walk from resolved target
    elif stat == exists:
      resolved = candidate
    elif stat == ENOENT:
      resolved = join(resolved, remaining segments)  # path not yet created; OK
      break
    else:
      return error
  return resolved
```

After canonicalization, MUST assert `canonical_workspace` has `canonical_root + "/"` as a prefix. If it does not, reject with `ErrWorkspaceSymlinkEscape`.

For remote workers, the same check MUST be performed via a shell script that resolves each path segment before `mkdir`.

### 11.3 Workspace lifecycle

1. `after_create` — runs once when the workspace directory is first created.
2. `before_run` — runs before every attempt. Fatal if it fails.
3. `after_run` — runs after every attempt (success or failure). Best-effort.
4. `before_remove` — runs before the workspace is deleted.

All hooks run in the workspace directory as the working directory.

### 11.4 Local workspace creation

```
ensure_workspace(workspace):
  if directory exists:
    return (workspace, created=false)
  if file exists at path:
    rm -rf path
  mkdir -p path
  return (workspace, created=true)
```

### 11.5 Remote workspace creation

For SSH workers, the orchestrator runs a shell script on the remote host that atomically creates and resolves the workspace, then echoes a tab-separated marker line:

```
printf '%s\t%s\t%s\n' '__SINGULARITY_WORKSPACE__' "$created" "$(pwd -P)"
```

The orchestrator parses this line from stdout to confirm the resolved canonical path.

---

## 12. Worktree Isolation

### 12.1 Modes

```toml
[harness]
worktree_mode = "branch-per-slice"   # or "milestone-per-worktree"
```

**`branch-per-slice`** (default):
- Each slice gets its own git branch (`sf/m{n}-s{n}-{slug}`) created from the current base.
- The harness emits `WorktreeCreate` before branch creation; the git service handles the actual `git worktree add`.
- After PostUnit hooks run, the git service merges the branch to the integration branch. The harness waits for the merge hook before marking the slice complete.
- Merge conflicts emit `MergeConflict`, which triggers `SignalPause`.

**`milestone-per-worktree`**:
- A single worktree created for the entire milestone.
- All slices share that worktree. The git service commits incrementally.
- The worktree is merged at milestone PostUnit time.

### 12.2 Rules

- The harness MUST emit `WorktreeCreate` and `WorktreeDelete` events. It MUST NOT call `git` directly.
- `worktree_mode` is session-immutable — changing it requires restart.

---

## 13. Verification Gates

### 13.1 Configuration

```toml
[harness.gates]
post_slice     = ["./gates/run-tests.sh", "./gates/lint.sh"]
post_milestone = ["./gates/integration-tests.sh"]
```

### 13.2 Execution rules

- Gates run as subprocesses. The `UnitResult` JSON is passed via stdin.
- Exit 0 = pass. Non-zero = fail.
- Fail increments the retry counter for that unit.
- Default max retries: 3. Configurable per gate type.
- On retry, the harness re-dispatches the same unit with gate failure output appended to context. The agent MUST see what failed and why.
- After max retries, the harness transitions to `PhaseReassess` and emits `GateBlocked` on pubsub.
- Gate results MUST be stored in `gate_results` table and written as span events on the unit span.

```go
type GateResult struct {
    GateName   string
    UnitID     string
    Passed     bool
    Attempt    int
    MaxRetries int
    Output     string        // combined stdout+stderr, truncated at 8KB
    Duration   time.Duration
}
```

### 13.3 PhaseReview — chunked review

Large diffs MUST NOT be reviewed in a single pass. The harness MUST split the changed file list into chunks of ≤ 300 lines (`ReviewChunkLines = 300`) before dispatching the review agent. Each chunk is reviewed independently; findings are accumulated. The final synthesis pass reviews all findings across chunks.

Files larger than `ReviewChunkLines` get their own chunk.

### 13.4 Unit archive

When a slice or milestone reaches `PhaseComplete`, the harness MUST move its artifact directory from `.sf/active/` to `.sf/archive/{YYYY-MM-DD}-{unit-id}/` atomically (rename, not copy+delete).

`.sf/active/` holds only in-progress work. `.sf/archive/` is queried by `/sf history`.

### 13.5 Reserved

(`specs.check`, godoc enforcement on the harness package, is a singularity-crush CI requirement — see § 1 — not a runtime gate against user projects.)

---

## 14. Configuration

### 14.1 File locations and precedence

1. `~/.sf/config.toml` — global defaults
2. `.sf/config.toml` — project overrides (takes precedence)

Both files are TOML. Project overrides global on a per-key basis.

### 14.2 Canonical schema

```toml
[harness]
context_compact_at    = 0.80
context_hard_limit    = 0.95
unit_timeout          = "10m"   # bounds one attempt: workspace creation through teardown
turn_timeout          = "5m"    # bounds one model turn
stall_timeout         = "2m"    # AttemptStalled when no agent event for this long
max_turns_per_attempt = 50
max_attempts          = 6       # exponential backoff before giving up
hot_cache_turns       = 10      # in-memory recent-turn buffer
supervisor_interval   = "10s"
max_retry_backoff     = "5m"
doc_sync              = true
turn_input_required   = "soft"  # or "hard"
worktree_mode         = "branch-per-slice"

[harness.concurrency]
max_agents                   = 10
max_agents_by_phase.execute  = 4
max_agents_by_phase.tdd      = 4
max_agents_by_phase.verify   = 10

[harness.auto_approve]
tools = ["bash:read", "fs:read", "git:status", "git:diff"]

[harness.hooks]
pre_dispatch = ["./hooks/pre-dispatch.sh"]
post_unit    = ["./hooks/post-unit.sh"]
after_create = "./hooks/after-create.sh"
before_run   = "./hooks/before-run.sh"
after_run    = "./hooks/after-run.sh"
before_remove = "./hooks/before-remove.sh"
timeout_ms   = 60000

[harness.gates]
post_slice     = ["./gates/run-tests.sh"]
post_milestone = ["./gates/integration-tests.sh"]

[harness.log]
path      = "~/.sf/log/sf.log"
max_size  = 10485760   # 10MB
max_files = 5
stderr    = false

[server]
port = 7842   # 0 = ephemeral (tests)

[worker]
ssh_hosts                      = []
max_concurrent_agents_per_host = 3

[routing]
research = "reasoning"
plan     = "reasoning"
execute  = "standard"
tdd      = "standard"
verify   = "fast"
review   = "standard"
merge    = "fast"
complete = "fast"
reassess = "reasoning"

[tiers.fast]
models = ["claude-haiku-4-5", "gemini-flash-2.0"]

[tiers.standard]
models = ["claude-sonnet-4-6", "gemini-2.0-pro"]

[tiers.reasoning]
models = ["claude-opus-4-7", "o3"]
```

### 14.3 Dynamic reload

The harness MUST poll `.sf/config.toml` on every orchestrator tick using a `{mtime, size, content_hash}` stamp. `content_hash` is SHA-256 of the file bytes.

When the stamp changes:
- Re-parse and re-validate.
- On success: apply changes immediately to future dispatch, concurrency limits, and hook lists. In-flight runs are NOT interrupted.
- On failure (parse error, validation error): log error at WARNING level, keep last known good config. MUST NOT crash.

The following fields are session-immutable even with dynamic reload enabled:
- `worktree_mode`
- `context_compact_at`
- `context_hard_limit`

Changing session-immutable fields requires restart.

### 14.4 Startup validation

The harness MUST validate config at startup and MUST fail fast with a descriptive error on invalid config. It MUST NOT silently ignore unknown keys or bad values. `/sf doctor` MUST run `HarnessConfig.Validate()` as one of its checks.

---

## 15. Model Routing

### 15.1 Three tiers

Each tier holds multiple candidate models. The router picks within the tier; it does not change the tier assignment.

### 15.2 Phase → tier mapping

Static, config-driven (see § 14.2 `[routing]` table). The harness MUST apply the phase-to-tier mapping before each dispatch. The agent MUST NOT influence this mapping.

The harness MUST set `Think: true` on the model config for phases mapped to `reasoning` tier.

### 15.3 Complexity upgrade

A classifier at dispatch time — file count, scope breadth, cross-cutting changes → complexity score. If the score crosses a configurable threshold, the tier bumps one level (fast→standard, standard→reasoning). The fingerprint and upgrade decision MUST be stored in SQLite for future routing decisions.

### 15.4 Within-tier selection

Within a tier, the router picks the model with the highest benchmark score:

```
score = quality * 0.6 + (1 - normalised_latency) * 0.2 + (1 - normalised_cost) * 0.2
```

Weights are configurable. If no benchmark data exists for the current fingerprint, use the tier's first model.

Models with a tripped circuit breaker (§ 9.3) MUST be skipped.

### 15.5 `/sf rate` feedback loop

Two signal sources:

- **Auto-mode** — the agent self-evaluates at unit close: `over` / `ok` / `under` relative to phase objective. No human in the loop.
- **Interactive mode** — human signals `over` / `ok` / `under` after reviewing unit output.

Both write to `benchmark_results`. Human ratings carry higher weight than LLM self-ratings (configurable multiplier, default 3×).

Score mappings: `over=0.3` (over-resourced), `ok=0.8`, `under=0.0` (blocks model for this fingerprint).

---

## 16. Knowledge Layer

### 16.1 Architecture

The knowledge layer is **Hindsight** ([vectorize-io/hindsight](https://github.com/vectorize-io/hindsight-cookbook)) — a memory service accessible on the tailnet. singularity-crush uses `hindsight-client-go`. There is no local vector store, no sqlite-vec table, no FTS5 fallback — all retrieval and persistence go through Hindsight.

SQLite in singularity-crush holds **orchestration state only** (sessions, units, blockers, gates, benchmarks, circuit breakers, agents). Memories, learnings, anti-patterns, and codebase context live in Hindsight.

When Hindsight is unreachable, the harness MUST log a warning and dispatch with no recall context. The agent still runs; it just lacks historical memory for that session. The harness MUST NOT block dispatch on Hindsight availability.

### 16.2 Memory tiers

Two tiers prevent token bloat during long-running sessions:

**Hot cache** — current dispatch's recent turns held in memory (never persisted to SQLite). Configurable size: `[harness] hot_cache_turns = 10`. Cleared on compaction.

**Hindsight store** — durable. PostUnit writes summaries, learnings, and anti-patterns. Pre-dispatch reads top-N most relevant entries. On compaction, the hot cache is summarised and written to Hindsight as a `session_summary` entry.

The harness MUST NOT mix the two tiers.

### 16.3 Two-bank pattern

Each session uses two Hindsight banks, queried separately and merged before each dispatch:

```go
projectRecall := hindsight.Recall("project/"+projectHash, query)
globalRecall  := hindsight.Recall("global/coding", query)
// merge, deduplicate, inject top-N into unit context
```

Concurrent `retain` calls from parallel slice workers use `document_id` derived from content hash. Duplicate memories silently overwrite rather than accumulate.

### 16.4 Anti-pattern library

Anti-patterns are memories tagged `collection: anti_patterns`, `is_negative: true`. They:
- Are written explicitly when the agent makes a mistake (gate failure or user feedback).
- MUST NOT be subject to normal maturation decay — they persist at full weight until explicitly removed.
- Are retrieved at dispatch time and presented in a dedicated block: `<anti_patterns>avoid these mistakes...</anti_patterns>`.

```go
type AntiPattern struct {
    ID          string
    Description string    // what went wrong
    Context     string    // when/where this applies
    CorrectPath string    // what to do instead
    SourceUnit  string
    CreatedAt   time.Time
}
```

### 16.5 Pattern maturation

| State | Condition | Retrieval weight |
|---|---|---|
| `candidate` | < 3 observations | 0.5× |
| `established` | ≥ 3 obs, harmful ratio < 30% | 1.0× |
| `proven` | decayed helpful score ≥ 5, harmful ratio < 15% | 1.5× |
| `deprecated` | harmful ratio > 30% | 0× (excluded) |

After 3 failed uses, content is prefixed `AVOID:` and flagged `is_negative: true`.

### 16.6 Confidence decay

```
halfLife    = 90 * (0.5 + confidence)   // days; confidence ∈ [0.0, 1.0]
decayFactor = 0.5 ^ (ageInDays / halfLife)
finalScore  = similarityScore * decayFactor
```

Memory access tiers: **hot** (accessed within 7 days), **warm** (within 30 days), **cold/stale** (older).

Entries with 10+ accesses gain a 7-day buffer against decay. Calling `validate()` when a memory directly aids task completion resets the decay timer.

### 16.7 Retrieval pipeline

Retrieval is delegated to Hindsight via `hindsight.Recall(bank, query, opts)`. Hindsight runs its own internal pipeline — fused semantic + lexical retrieval, optional reranking, and decay weighting — and returns ranked entries. The harness does not implement a retrieval pipeline of its own.

Recall options the harness uses:

| Option | Use |
|---|---|
| `top_k` | Number of entries to inject into prompt (default 5) |
| `bank` | `project/{hash}` or `global/coding` (§ 16.3) |
| `filter` | Tag filters (e.g. `collection=anti_patterns`) |
| `rerank_quality` | `fast` (routine) or `accurate` (pre-dispatch context injection) |

The harness applies its own maturity and anti-pattern weighting (§ 16.4, § 16.5) by tagging entries on retain and filtering / re-ordering on recall — Hindsight stores the metadata but does not interpret it.

### 16.8 `sf init`

Deep analysis is default, not opt-in:

1. AST-level codebase scan (languages, structure, entry points, dependencies).
2. Git history analysis (active areas, recent changes, contributors).
3. Retain findings into `project/{hash}` Hindsight bank.
4. Establish `.sf/config.toml` with detected stack, workflow templates, model routing hints.

`--quick` flag skips Hindsight indexing for throwaway sessions.

---

## 17. Persistent Agents

### 17.1 Agent vs unit

A **unit** is ephemeral work pulled from a tracker (§ 3.3) and driven through the phase state machine (§ 4). It is archived on completion.

A **persistent agent** is a named, long-lived identity: it has its own memory blocks, system prompt, and message history. It sleeps at zero cost when idle and wakes when its inbox receives a message or an explicit `/sf agent run <name>` is issued.

**A persistent agent run is NOT a unit.** Specifically:

| Aspect | Unit | Persistent agent run |
|---|---|---|
| Source of work | Tracker (§ 3.3) | Inbox message or explicit `/sf agent run` |
| Phase state machine | YES | NO |
| Verification gates | YES | NO |
| Workflow templates | YES | NO |
| PostUnit hooks | YES | NO (replaced by `PostAgentRun`) |
| `before_run` / `after_run` workspace hooks | YES | YES (shared lifecycle) |
| Supervisor checks (StuckLoop, AbandonDetect, BudgetWarning) | YES | YES |
| Crash recovery | re-dispatch from last phase | re-deliver undelivered inbox |
| Budget instance | fresh per attempt | persistent across runs (until reset) |

What they share: the worker attempt lifecycle (§ 6) — workspace creation, `before_run` hook, agent session, turn loop, `after_run` hook — is identical. The supervisor goroutine monitors agent runs and unit attempts with the same checks. The trace records both as runs with distinct `run_kind` attributes.

### 17.2 Memory block injection

At dispatch time, the harness MUST render the agent's memory blocks into the system prompt:

```xml
<memory>
  <block label="persona">{{value}}</block>
  <block label="human">{{value}}</block>
  <block label="task">{{value}}</block>
</memory>
```

### 17.3 Built-in memory tools

| Tool | Signature | Effect |
|---|---|---|
| `core_memory_append` | `(label string, content string)` | Appends content to block, respects `char_limit` |
| `core_memory_replace` | `(label string, old string, new string)` | Replaces substring in block |

Both tools MUST write to `agent_memory_blocks` in SQLite before the next turn is dispatched. A crash mid-session MUST preserve the updated block state.

### 17.4 Agent lifecycle

```go
type AgentState int

const (
    AgentIdle    AgentState = iota // no pending messages, not running
    AgentRunning                   // dispatched, consuming tokens
    AgentWaiting                   // sent a message to another agent, awaiting reply
    AgentStopped                   // explicitly stopped; will not wake automatically
)
```

The harness owns all state transitions. The agent loop MUST NOT write `AgentState` directly.

### 17.5 Agent fleet supervision

Each persistent agent has its own `Budget` instance (§ 8) that persists across runs and is reset only on explicit `/sf agent reset <name>`. Compaction fires per-agent — when one agent's budget hits the compact threshold, only its hot cache is summarised; other agents are unaffected.

Crash recovery for agents differs from unit recovery (§ 4.7): on restart, each agent's `agent_inbox` is rescanned for `delivered = 0` rows. Any such rows trigger an immediate `AgentWake` — the agent resumes processing the queue. There is no phase to resume; the inbox IS the resumption state.

The trace records each agent run as a separate root span with `run_kind = "agent"` and `agent_id = <id>`. `/sf session-report` breaks down spend by agent.

---

## 18. Inter-Agent Messaging

### 18.1 `send_message` tool

```go
// Tool the agent calls:
// send_message(to: string, message: string) -> void
//
// to:      agent name or agent ID
// message: plain text; the receiving agent sees it as a "user" role message
```

When called, the harness MUST:
1. Insert a row into `agent_inbox` for the target agent.
2. Emit an `AgentWake` pubsub event for the target agent.
3. Record the message in `agent_messages` for both sender and receiver.

### 18.2 Wake rules

- An `AgentIdle` agent that receives `AgentWake` MUST start a new dispatch cycle immediately.
- An `AgentRunning` agent queues the message for its next dispatch cycle.
- Undelivered inbox messages MUST be prepended to the context as `user` role messages in arrival order at the start of each dispatch, then marked `delivered = 1`.

### 18.3 `wait_for_reply`

An agent calling `wait_for_reply(ticket_id)` transitions to `AgentWaiting`. The harness suspends its dispatch loop until the target agent sends a reply or a configurable timeout elapses.

`wait_for_reply` has a mandatory timeout. The harness MUST NOT block indefinitely.

### 18.4 Agent handoff

`handoff(to, context)` transfers the active task to a named specialist agent:

1. The calling agent's current unit is suspended (not completed).
2. The target agent receives the full task context (system prompt, memory blocks, last N messages) pre-loaded in its inbox.
3. The calling agent transitions to `AgentWaiting` until the specialist replies.
4. If the target agent is not found or is `AgentStopped`, `handoff` returns an error and the calling agent continues.

```go
// Tool the agent calls:
// handoff(to: string, context: string) -> HandoffTicket
// Agent calls wait_for_reply(ticket.id) to block until the specialist responds.
```

### 18.5 Append-only inbox log

`agent_inbox` MUST be append-only. Rows MUST NOT be deleted after insert. `delivered` is the only mutable column. This gives a complete audit trail of all inter-agent communication.

Conflict resolution for shared state happens at read time (last-writer-wins on `agent_memory_blocks` by `updated_at`), not via locking.

### 18.6 What not to build

- **Shared memory** — agents MUST NOT share memory blocks. If two agents need a common fact, one sends it as a message.
- **Broadcast** — there is no `send_message_all`. Routing MUST be explicit.
- **Synchronous RPC** — `send_message` is fire-and-forget. `wait_for_reply()` is explicit and has a timeout.

---

## 19. Observability

### 19.1 Structured log format

All harness log lines MUST use stable `key=value` pairs. Required context fields:

| Scope | Required fields |
|---|---|
| Any unit-related log | `unit_id=`, `unit_type=` |
| Agent session lifecycle | `session_id=`, `turn_count=` |
| Phase transitions | `from=`, `to=`, `reason=` |
| Gate execution | `gate=`, `attempt=`, `passed=` |

Include action outcome in the message: `completed`, `failed`, `retrying`, `canceled`. MUST NOT log large raw payloads — truncate hook output at 2 KB and append `(truncated)`.

### 19.2 Log rotation

- Max file size: 10 MB.
- Max rotating files: 5.
- Single-line format — no multi-line log entries.
- When file logging is configured, the default stderr handler MUST be removed (logs to file only).
- Default path: `~/.sf/log/sf.log`.

### 19.3 Spans and trace

```go
type Span struct {
    TraceID   string
    SpanID    string
    Operation string        // "tool_call" | "phase_transition" | "model_request" | "hook"
    StartedAt time.Time
    Duration  time.Duration
    Attrs     map[string]any
    Error     error
}
```

- Every tool call, phase transition, model request, and hook execution MUST emit a span.
- Spans MUST be written to `~/.sf/trace.jsonl` (rolls daily).
- Span emission MUST be non-blocking — use a buffered channel with a background writer goroutine.
- MUST NOT drop spans. If the buffer is full, block briefly rather than discard.

### 19.4 Intent chapters

Spans are grouped into named chapters by intent (not just by phase).

```go
type Chapter struct {
    ID       string
    UnitID   string
    Name     string        // inferred or agent-declared
    Intent   string        // one-sentence summary written at close
    OpenedAt time.Time
    ClosedAt *time.Time
    Outcome  string        // "success" | "failure" | "pivot"
    SpanIDs  []string
}
```

Chapters serve two purposes:
1. **Context recovery** — on resume after a crash, the harness reconstructs "what the agent was doing and why" from the chapter log. The chapter summary is injected at the top of the restored context.
2. **Hindsight recall** — completed chapters are stored as discrete memory units. Recall queries match against chapter intent.

The agent MAY open a chapter explicitly via `chapter_open(name)`.

### 19.5 HTTP observability API

The harness MUST expose a lightweight HTTP server on `localhost` when `server.port` is configured. The API is observability-only — orchestrator correctness MUST NOT depend on it.

**`GET /api/v1/state`** — runtime snapshot:

```json
{
  "generated_at": "2026-04-29T14:22:00Z",
  "counts": { "running": 3, "retrying": 1, "queued": 5 },
  "running": [
    {
      "unit_id": "execute-task/m1/s2/t3",
      "phase": "execute",
      "session_id": "sess-abc-turn-4",
      "turn_count": 7,
      "last_event": "tool_call",
      "started_at": "2026-04-29T14:10:00Z",
      "tokens": { "input": 18200, "output": 2100, "total": 20300 }
    }
  ],
  "retrying": [
    {
      "unit_id": "execute-task/m1/s2/t4",
      "attempt": 2,
      "due_at": "2026-04-29T14:24:00Z",
      "error": "gate: tests failed"
    }
  ],
  "totals": {
    "input_tokens": 84000,
    "output_tokens": 12000,
    "cost_usd": 1.24,
    "seconds_running": 4820
  }
}
```

**`GET /api/v1/units/<unit_id>`** — per-unit debug detail: recent events, workspace path, retry count, last error, log file path.

**`POST /api/v1/refresh`** — queue an immediate poll + reconciliation cycle (202 Accepted; best-effort coalescing of rapid requests).

### 19.6 Rate-limit tracking

The harness MUST track the latest rate-limit payload from any provider event and surface it in the TUI and HTTP API. Rate-limit data is observability-only — no retry logic is driven by it. Circuit breaker and backoff handle retries separately.

---

## 20. Failure Taxonomy

Every harness failure has a class. The class determines recovery behavior.

| Class | Examples | Recovery |
|---|---|---|
| `config` | Missing or invalid `.sf/workflows/*.toml`, invalid `.sf/config.toml`, unknown tracker kind, missing API key | Block new dispatches. Keep service alive. Continue reconciliation. Emit operator-visible error. |
| `workspace` | Directory creation failure, hook timeout, invalid path | Fail the current attempt. Orchestrator retries with backoff. |
| `agent_session` | Startup handshake failed, turn timeout, turn cancelled, subprocess exit, stalled session, `turn_input_required` (hard mode) | Fail the current attempt. Orchestrator retries with backoff. |
| `tracker` | API transport error, non-200, GraphQL errors, malformed payload | **Candidate fetch failure**: skip this tick. **Reconciliation failure**: keep workers running, retry next tick. |
| `observability` | Snapshot timeout, dashboard render error, log sink failure | Log and ignore. MUST NOT crash the orchestrator over an observability failure. |

### 20.1 Typed error codes

```go
const (
    ErrMissingWorkflowFile      = "missing_workflow_file"   // .sf/workflows/<name>.toml not found
    ErrWorkflowParseError       = "workflow_parse_error"
    ErrUnsupportedTrackerKind   = "unsupported_tracker_kind"
    ErrMissingTrackerKey        = "missing_tracker_api_key"
    ErrWorkspaceCreation        = "workspace_creation_failed"
    ErrWorkspaceSymlinkEscape   = "workspace_symlink_escape"
    ErrHookTimeout              = "hook_timeout"
    ErrHookFailed               = "hook_failed"
    ErrAgentStartup             = "agent_session_startup"
    ErrTurnTimeout              = "turn_timeout"
    ErrTurnFailed               = "turn_failed"
    ErrTurnInputRequired        = "turn_input_required"
    ErrPromptRender             = "prompt_render_failed"
    ErrBudgetExhausted          = "budget_exhausted"
    ErrStalled                  = "stalled"
    ErrCanceledByReconciliation = "canceled_by_reconciliation"
    ErrModelUnavailable         = "model_unavailable"
    ErrCircuitOpen              = "circuit_open"
)
```

Implementations MUST match on typed error codes. Matching on error message strings is PROHIBITED.

### 20.2 Scheduler state

Scheduler state is intentionally in-memory. Restart recovery MUST NOT attempt to restore retry timers, live sessions, or in-flight agent state. After restart: startup terminal cleanup → fresh poll → re-dispatch eligible work. This is a design choice, not a limitation. Durable retry state is a future extension.

---

## 21. Trust Boundary

Every deployment MUST document its trust posture explicitly. There is no universal safe default.

### 21.1 Default posture (single-user developer machine)

- Auto-approve tool execution and file changes within the workspace.
- `turn_input_required = "soft"`.
- Workspace isolation enforced (symlink-aware path containment, sanitized names).
- Secrets from Vault only — MUST NOT store secrets in config files in plaintext.

### 21.2 Hardening measures for less-trusted environments

- Filter which issues/tasks are eligible for dispatch — untrusted or out-of-scope tasks MUST NOT automatically reach the agent.
- Restrict the tracker client-side tool to read-only or project-scoped mutations only.
- Run the agent subprocess under a dedicated OS user with no write access outside the workspace root.
- Add container or VM isolation around each workspace (Docker, nsjail, etc.).
- Restrict network access from the workspace.
- Narrow available tools to the minimum needed for the workflow.

### 21.3 Auto-approval contract

In auto-mode the harness calls `permission.AutoApproveSession()` ONLY for operations listed in `[harness.auto_approve]`. Sensitive operations (`fs:write-outside-project`, `shell:exec`) MUST always prompt regardless of auto-mode setting.

SF-specific permission gates:
- `git:write` — any git operation that mutates state. Requires explicit grant in auto-mode.
- `worktree:create` and `worktree:delete` — worktree lifecycle.
- `fs:write-outside-project` — ALWAYS prompt, NEVER auto-approve.
- `shell:exec` — allowlist specific commands; no blanket approval.

---

## 22. Distributed Execution

### 22.1 Topology

The orchestrator ALWAYS runs centrally. Workers MAY execute on remote hosts over SSH.

```toml
[worker]
ssh_hosts                      = ["mikki-bunker", "forge-gpu-1"]
max_concurrent_agents_per_host = 3
```

### 22.2 Rules

- `workspace.root` is resolved on the **remote host**, not the orchestrator.
- The agent subprocess is launched over SSH stdio. The orchestrator owns the session lifecycle.
- Continuation turns within one worker lifetime MUST stay on the same host and workspace.
- If a host is at capacity, dispatch MUST wait rather than silently fall back to local or another host.
- Once a run has produced side effects, moving to another host on retry is treated as a new attempt (not invisible failover).
- The run record MUST include `worker_host` so operators can see where each run executed.
- SSH workspace creation MUST use the same symlink-aware validation as local workspaces, implemented via shell script.

---

## 23. Plugin Extension Points

Plugin interfaces are activated when Crush's compile-time Go plugin system stabilises.

### 23.1 Interfaces

**`SupervisorCheck`** — custom supervisor checks without forking:

```go
type SupervisorCheck interface {
    Name() string
    Check(ctx context.Context, state SupervisorState) SupervisorSignal
}
```

**`Shipper`** — PR/MR creation. GitHub default; GitLab, Gitea, Forgejo alternatives:

```go
type Shipper interface {
    Ship(ctx context.Context, opts ShipOptions) (ShipResult, error)
}
```

**`VCS`** — version control backend. `git` default; `jj` (Jujutsu) first alternative:

```go
type VCS interface {
    Commit(ctx context.Context, msg string, files []string) error
    Branch(ctx context.Context, name string) error
    Push(ctx context.Context, remote, branch string) error
}
```

**`Store`** — storage backend. SQLite for personal use; PostgreSQL for team sessions:

```go
type Store interface {
    SaveSession(ctx context.Context, s Session) error
    LoadSession(ctx context.Context, id string) (Session, error)
    SaveMemory(ctx context.Context, m Memory) error
    SearchMemory(ctx context.Context, q MemoryQuery) ([]Memory, error)
}
```

**`Notifier`** — notification provider. Slack, Discord, webhook:

```go
type Notifier interface {
    Notify(ctx context.Context, event Event) error
}
```

### 23.2 What stays out of plugins

- Workflow templates — enforced TOML/YAML data
- Skills — `SKILL.md` prompt guidance
- Model routing — config + SQLite + thin Go scorer
- Phase transitions — harness-owned, not extensible

---

## 24. Secret Management

### 24.1 `vault://` URI scheme

Secrets MUST NOT be stored in config files in plaintext. The canonical secret reference format is:

```
vault://secret/singularity-crush#anthropic_api_key
```

In config:

```json
{
  "providers": {
    "anthropic": { "api_key": "vault://secret/singularity-crush#anthropic_api_key" }
  }
}
```

### 24.2 VaultResolver

```go
type VaultResolver struct {
    client *vault.Client
}

func (r *VaultResolver) Resolve(uri string) (string, error) {
    // parse vault://path#field
    // client.KVv2(mount).Get(ctx, path) → secret.Data["field"]
}
```

Auth chain (first that succeeds):
1. `VAULT_TOKEN` env var (CI / ephemeral)
2. `~/.vault-token` file (local dev)
3. AppRole via `VAULT_ROLE_ID` + `VAULT_SECRET_ID` (production)

Secrets MUST be fetched once at startup and held in memory for the session lifetime. MUST NOT be written to disk or logged.

### 24.3 Stopgap

Until the native resolver is built, use Crush's existing `$(command)` substitution:

```json
{ "api_key": "$(vault kv get -field=anthropic_api_key secret/singularity-crush)" }
```

---

## 25. CLI Commands

### `/sf auto`

Start the autonomous loop. The harness polls for eligible units and dispatches workers until no more eligible units exist or until stopped by `/sf pause`.

### `/sf next`

Manual step mode. Dispatch one unit, wait for completion, surface result. Repeat on each invocation.

### `/sf dispatch <unit-id>`

Force-dispatch a specific unit regardless of priority or blocker state. Surfaces a warning if blockers exist.

### `/sf pause`

Cleanly pause auto-mode. Writes `SessionPaused` to SQLite. All in-flight units complete their current turn before stopping.

### `/sf status`

Structured project health snapshot:

```
Project: singularity-foundry
Phase:   Execute  [m2/s3/t1 — add trace export]
Next:    TDD      [m2/s3/t1]
Blocker: none

Milestones:  2 / 5  (40%)
Slices:      7 / 18 (39%)
Tasks:       14 / 42 (33%)

Session:  4h 12m  |  $0.83  |  claude-sonnet-4-6
```

Blockers surface from the `session_blockers` table. `/sf status` MUST NOT poll pubsub — it reads SQLite directly.

### `/sf revert <unit-id>`

Four-phase git-aware revert protocol:

1. **Target selection** — accept explicit unit ID, or present the top 3 in-progress + 3 most recent completed units as a numbered menu.
2. **Git reconciliation** — find all commits belonging to the target unit. Handle ghost commits (SHA missing after rebase/squash) by searching by commit message prefix.
3. **Confirmation** — display exact SHA list with descriptions and dates. Warn on merge commits.
4. **Execution** — `git revert --no-edit <sha>` in reverse order (newest first). On conflict: `SignalPause`.

After all reverts: restore `.sf/active/{unit-id}/` artifacts from archive; mark unit as `[ ]` in the plan.

### `/sf rate over|ok|under`

Signal model quality for the most recent unit. Writes to `benchmark_results` with human-rating weight multiplier.

### `/sf benchmark`

Run on-demand model benchmarks for all tiers against real task samples. Updates `benchmark_results`.

### `/sf doctor`

Run health checks:
- `HarnessConfig.Validate()`
- Vault connectivity
- Hindsight connectivity
- SQLite schema version
- Lock file state
- `specs.check` on harness package
- Workflow template syntax

### `/sf history`

Query archived units in `.sf/archive/`. Supports filtering by date, phase, model, verdict.

### `/sf forensics`

Inspect the trace for a specific unit or session. Shows all spans, tool calls, phase transitions, and gate results in chronological order.

### `/sf reset-circuits`

Clear all tripped circuit breakers. Next dispatch uses benchmark scores to select within each tier normally.

---

## 26. Conformance Checklist

Use this checklist as the definition-of-done for each build phase. An implementation is **core-conformant** when all core items pass. **Extension-conformant** when all extension items also pass.

### 26.1 Core (must ship)

- [ ] **C-01** Workflow template TOML loader with `phases`, `require_tdd`, `require_review`, `max_retries`, `max_reassess` fields; unknown fields rejected.
- [ ] **C-02** Phase state machine with all 10 phases; invalid transitions rejected with typed error at harness boundary.
- [ ] **C-03** `Harness.Transition(ctx, from, to, reason)` persists to SQLite before new phase begins; emits pubsub `PhaseChange` after write.
- [ ] **C-04** AttemptState enum (11 states); `AttemptCanceled` distinct from `AttemptFailed`.
- [ ] **C-05** TurnKind enum; continuation turns receive guidance-only prompt, not full task prompt.
- [ ] **C-06** Strict prompt rendering: unknown `{{variable}}` in template → startup panic.
- [ ] **C-07** `attempt` variable: `null` on first dispatch; integer ≥ 1 on retry; `last_error` auto-injected on retry.
- [ ] **C-08** `turn_input_required` configurable `soft` (inject non-interactive message) or `hard` (fail immediately); MUST NOT stall indefinitely.
- [ ] **C-09** Context budget: `ShouldCompact()` triggers compaction before next turn; `AtHardLimit()` halts unit; budget state persisted to SQLite after every turn.
- [ ] **C-10** Budget token accounting prefers absolute totals; prevents double-counting.
- [ ] **C-11** Compaction: write session summary to Hindsight, clear hot cache, start next turn with fresh Hindsight recall.
- [ ] **C-12** Supervisor goroutine: all 9 built-in checks; communicates only via pubsub; MUST NOT call `os.Exit`.
- [ ] **C-13** Circuit breaker: 3 consecutive non-transient failures trips model; state persisted to SQLite; resets after 24h or `/sf reset-circuits`.
- [ ] **C-14** `ModelUnavailable` → `SignalAbort` immediately (not after timeout).
- [ ] **C-15** Hook events: `PreDispatch`, `PostUnit`, `PhaseChange`, `AutoLoop`, `WorktreeCreate`, `WorktreeDelete`, `MergeReady`, `MergeConflict`.
- [ ] **C-16** `UnitResult` struct passed to PostUnit hooks as JSON via stdin.
- [ ] **C-17** PostUnit hooks run sequentially; non-zero exit → `SignalAbort`; timeout → kill, log, continue.
- [ ] **C-18** Tool response contract: `{success, output, contentItems}` shape for all tool responses.
- [ ] **C-19** Unknown tool call → structured failure response; session continues.
- [ ] **C-20** Doc sync hook runs after every `PhaseMerge`; MAY be disabled with `doc_sync = false`.
- [ ] **C-21** Workspace name sanitization: `[^a-zA-Z0-9._-]` → `_`.
- [ ] **C-22** Symlink-aware workspace path containment via segment-by-segment `lstat` canonicalization; naive `EvalSymlinks` is insufficient.
- [ ] **C-23** Workspace lifecycle hooks: `after_create`, `before_run`, `after_run`, `before_remove`; `before_run` fatal, `after_run` best-effort.
- [ ] **C-24** Startup cleanup: stale active artifacts moved to archive; running units marked interrupted.
- [ ] **C-25** Dynamic config reload: `{mtime, size, SHA-256}` stamp polled every tick; invalid reload keeps last known good; session-immutable fields unchanged without restart.
- [ ] **C-26** Per-phase concurrency caps (`max_agents_by_phase`).
- [ ] **C-27** Blocker-aware dispatch: non-terminal upstream → skip, re-evaluate next tick; no backoff increment.
- [ ] **C-28** Priority sort: priority asc → blocker-free first → phase order → created_at asc → id lexicographic.
- [ ] **C-29** Continuation retry (1s) after normal worker exit.
- [ ] **C-30** Exponential backoff after abnormal exit; cap configurable (default 5m).
- [ ] **C-31** Structured log format: `key=value` pairs; required context fields per scope; truncate at 2KB.
- [ ] **C-32** Log rotation: 10MB max, 5 files, single-line format, stderr handler removed when file logging active.
- [ ] **C-33** Span-based trace to `~/.sf/trace.jsonl`; non-blocking buffered writer; MUST NOT drop spans.
- [ ] **C-34** Intent chapters: open/close with intent summary; used for crash recovery context and Hindsight recall.
- [ ] **C-35** Typed error codes; matching on error strings PROHIBITED.
- [ ] **C-36** Scheduler state intentionally in-memory; restart re-dispatches from fresh poll.
- [ ] **C-37** Project CI runs `specs.check`: AST-based godoc enforcement on all exported identifiers in singularity-crush's own harness packages. (Not a user-project runtime gate.)
- [ ] **C-38** Vault secret resolution: `vault://path#field` URI scheme; auth chain: `VAULT_TOKEN` → `~/.vault-token` → AppRole; secrets MUST NOT be written to disk or logged.
- [ ] **C-39** PhaseReview chunked at ≤ 300 lines per chunk.
- [ ] **C-40** Unit archive: `.sf/active/` → `.sf/archive/{date}-{unit-id}/` on `PhaseComplete` via atomic rename.
- [ ] **C-41** Tracker interface: `Kind`, `FetchCandidates`, `FetchUnitState`, `UpdateUnitState`; status enum `active | blocked | done | cancelled | unknown`; built-in `linear`, `github`, `jira`, `sqlite` adapters; `(tracker_kind, tracker_id)` is the unique upstream key.
- [ ] **C-42** Tracker failures never crash the orchestrator: candidate-fetch failure skips tick; state-fetch failure mid-attempt logs and continues turn loop.
- [ ] **C-43** Crash recovery: `running` units → `interrupted` on startup; re-dispatch fresh from last persisted phase boundary with `last_error = "resumed_after_crash"`; tool calls NOT replayed; agent sessions NOT resumed.
- [ ] **C-44** Process lock at `~/.sf/run.lock`; stale-lock cleanup via `/proc` PID check.
- [ ] **C-45** Doc-sync runs as a sub-step of `PhaseMerge` (not a separate phase, not a post-merge dispatch); empty diff is a no-op; user approval required unless `doc_sync_auto_approve = true`.
- [ ] **C-46** SQLite is orchestration-only — no `memories` table, no vector index. Knowledge MUST live in Hindsight.

### 26.2 Knowledge layer (ship after core)

- [ ] **K-01** Memory tiers: hot cache (in-memory, last 10 turns); Hindsight store (durable, PostUnit writes).
- [ ] **K-02** Two-bank Hindsight pattern: `project/{hash}` + `global/coding`; merged before dispatch.
- [ ] **K-03** Anti-pattern library: `collection: anti_patterns`; never decay; surfaced in dedicated `<anti_patterns>` block.
- [ ] **K-04** Pattern maturation: 4 states (candidate → established → proven → deprecated); weights as specified.
- [ ] **K-05** Confidence decay: `halfLife = 90 * (0.5 + confidence)` days.
- [ ] **K-06** Hindsight is the sole knowledge backend; on Hindsight outage, dispatch proceeds with empty recall and a logged warning.
- [ ] **K-07** `sf init` deep analysis default; `--quick` skips Hindsight indexing.

### 26.3 Model routing (ship after core)

- [ ] **R-01** Three tiers; phase → tier static mapping from config.
- [ ] **R-02** `Think: true` set for `reasoning` tier phases; agent cannot override.
- [ ] **R-03** Within-tier selection by benchmark score formula.
- [ ] **R-04** Complexity upgrade: classifier at dispatch time; fingerprint stored in SQLite.
- [ ] **R-05** `/sf rate` writes `benchmark_results`; human ratings carry 3× weight.

### 26.4 Persistent agents (ship after core)

- [ ] **A-01** `agents`, `agent_memory_blocks`, `agent_messages`, `agent_inbox` SQLite tables.
- [ ] **A-02** Memory block injection as XML into system prompt at dispatch.
- [ ] **A-03** `core_memory_append` and `core_memory_replace` tools write to SQLite before next turn.
- [ ] **A-04** `AgentState` enum (4 states); harness owns all transitions.
- [ ] **A-05** `agent_inbox` append-only; `delivered` is the only mutable column.
- [ ] **A-06** `send_message` tool: inserts to inbox, emits `AgentWake`.
- [ ] **A-07** `wait_for_reply` with mandatory timeout; MUST NOT block indefinitely.
- [ ] **A-08** `handoff(to, context)`: suspends calling agent → target receives full context → calling agent transitions to `AgentWaiting`.
- [ ] **A-09** Per-agent budget tracking, supervision, and crash recovery.
- [ ] **A-10** Cost recorded per agent in trace.

### 26.5 Extensions (ship after core)

- [ ] **E-01** HTTP observability API: `GET /api/v1/state`, `GET /api/v1/units/<id>`, `POST /api/v1/refresh`.
- [ ] **E-02** SSH worker extension: `worker.ssh_hosts`; remote workspace creation via shell script with symlink-aware validation; per-host concurrency cap.
- [ ] **E-03** Durable retry queue across restarts (SQLite-backed).
- [ ] **E-04** `tracker_query` client-side tool (agent reads/updates task state via orchestrator auth).
- [ ] **E-05** Plugin interfaces: `SupervisorCheck`, `Shipper`, `VCS`, `Store`, `Notifier`.

---

*End of SPEC.md v0.1.0-draft*
