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

singularity-crush is a **fork** of Crush, distributed as a single binary named `sf`. It re-uses Crush's `internal/` packages directly. Go's `internal/` visibility rule means only code in the same module tree may import `internal/...` — the fork model satisfies this constraint cleanly: singularity-crush's `harness/`, `tracker/`, etc. live alongside Crush's `internal/` in the same module (`github.com/singularity-ng/crush`). Vendoring or external imports would not work. The TUI's slash-command router is extended with the `/sf <subcommand>` namespace (§ 25); existing Crush slash commands (`/help`, `/clear`, etc.) continue to work unchanged. This means an `sf` user can drive interactively like vanilla Crush, then opt into autopilot via `/sf auto`.

### 1.1 Versioning

singularity-crush follows [SemVer 2.0](https://semver.org/). For this spec:

- **Patch** (0.x.Y): clarifications, conformance refinements, no behavioural change.
- **Minor** (0.Y.0): additions to the harness API, schema, or CLI that do not break existing implementations.
- **Major** (X.0.0): breaking changes to schema, hook contracts, or harness API. v1.0 is reserved for the first non-draft release.

While the spec is on `0.x.y-draft`, all sections are subject to change. v1.0.0 freezes §§3 (Data Model), 4 (Phase State Machine), 6 (Worker Attempt Lifecycle), 10 (Hooks), 14 (Configuration), and 26 (Conformance) — changes to those sections post-v1 require a major bump. It MUST NOT rebuild what Crush already provides:

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

Unit IDs use the format `{type}/{slug}` where slug is hierarchical:
- Milestone: `milestone/m{n}` (e.g. `milestone/m2`)
- Slice: `slice/m{n}/s{n}` (e.g. `slice/m2/s3`)
- Task: `task/m{n}/s{n}/t{n}` (e.g. `task/m2/s3/t1`)

The slug encodes the parent hierarchy redundantly with `units.parent_id` to make trace and log lines self-describing without requiring a join.

**Phase** — a named stage of a unit's lifecycle. The harness owns all phase transitions; no other layer may transition a phase directly.

**Attempt** — one dispatch of a worker for a unit. A unit may accumulate multiple attempts across failures and retries.

**Turn** — one model call within an attempt. An attempt consists of one or more turns. The first turn receives the full task prompt; subsequent turns receive continuation guidance only.

**Project** — a directory with `.sf/config.toml`. The project root is the directory containing `.sf/`. Each project has its own SQLite DB at `<project>/.sf/sf.db` — `~/.sf/sf.db` is the cross-project default DB used only when no project-local DB exists. Multiple projects on the same machine MUST use separate `.sf/` directories and therefore separate DBs, locks, and trace files.

**Session** — a top-level container scoped to one project, with a stable ULID, persisting across process restarts of the same project. A session is created on the first `/sf auto` or `/sf next` invocation in a project and reused on subsequent invocations until explicitly ended (`/sf session end`) or until 30 days of inactivity. The session holds the running state for all units, the context budget, and the supervisor state.

**Harness** — the layer between the agent loop (fantasy) and SF's orchestration logic (milestones, phases, git, worktrees). It owns: context budget, phase transitions, unit lifecycle hooks, session contract, observability, and supervision. Nothing in the planning or git layers MUST reach past the harness boundary into fantasy directly.

**Worker** — the process (local or SSH-remote) that executes one attempt. Spawned by the orchestrator.

**Orchestrator** — central process. Owns the scheduling loop, in-memory state, and all SQLite writes. Always runs locally even in distributed deployments.

**Singularity Memory** (`sm`) — the durable knowledge layer. An HTTP + MCP server holding memories, learnings, and anti-patterns across sessions, projects, and tools. Originally derived from `vectorize-io/hindsight` (MIT) and assimilated into our codebase under `singularity_memory_server/`; we own the engine. Runs either embedded (in-process for single-user sf) or remote (shared service on tailnet, reachable from sf, Hermes, OpenClaw, Claude Code, Cursor, etc.). Not SQLite — knowledge lives in Singularity Memory; SQLite holds only orchestration state.

**Skill** — a `SKILL.md` file providing prompt guidance to the agent. Inspirational, not enforced.

**Workflow template** — a TOML file specifying the exact phase sequence the harness enforces for a class of work. Programmatic, not a suggestion to the agent.

**Tracker** — the external system from which work is pulled (Linear, GitHub Issues, Jira, or built-in SQLite). The tracker is authoritative for unit status; the local `units` row mirrors it. See § 3.3.

**Claim** — a soft lock recorded on a `units` row indicating the orchestrator is currently dispatching it. Stored as `claim_holder` (worker host or PID) and `claim_until` (UNIX ms expiry). A claim is released on terminal phase, worker exit, or claim expiry. Prevents two orchestrators or two workers picking up the same unit simultaneously. The orchestrator MUST sweep expired claims at the start of every poll tick: any row with `claim_until < now()` and `phase_status = 'running'` is reset to `phase_status = 'interrupted'` and `claim_holder = NULL`.

**Run** — the unifying abstraction for one execution of the worker attempt lifecycle (§ 6). A run is either a **unit attempt** (driven by the phase state machine) or a **persistent agent run** (driven by inbox messages). The `runs` table (§ 3.5) records both, distinguished by `run_kind`. Trace, billing, and supervisor monitoring all key on `run_id`.

---

## 3. Data Model

The orchestrator uses a single SQLite database **per project** at `<project>/.sf/sf.db` (or `~/.sf/sf.db` for non-project sessions) for **orchestration state only**: sessions, units, phase transitions, blockers, gate results, benchmarks, circuit breakers, and persistent agents. **Knowledge** (memories, learnings, anti-patterns, codebase context) lives in Singularity Memory (§ 16), not SQLite.

All primary keys for runtime-allocated rows (sessions, units, runs, agents, agent_messages, agent_inbox, gate_results, session_blockers, pending_retain) MUST be [ULIDs](https://github.com/ulid/spec) — sortable by creation time without a separate timestamp column. Schema-natural keys (model name, agent name) remain TEXT but are not ULIDs.

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
    id           TEXT PRIMARY KEY,                    -- format: type/m{n}[/s{n}[/t{n}]]
    session_id   TEXT NOT NULL REFERENCES sessions(id),
    parent_id    TEXT REFERENCES units(id),           -- NULL for milestone; ≤ 3 levels deep
    type         TEXT NOT NULL CHECK (type IN ('milestone', 'slice', 'task')),
    workflow     TEXT NOT NULL,                       -- workflow template name; pinned at first dispatch
    workflow_hash TEXT NOT NULL,                       -- SHA-256 of pinned template content (FK workflow_pins.hash)
    phase        TEXT NOT NULL,
    phase_status TEXT NOT NULL CHECK (phase_status IN
                   ('pending', 'running', 'succeeded', 'failed', 'canceled', 'interrupted')),
    attempt      INTEGER NOT NULL DEFAULT 1,          -- 1 = first try, 2 = first retry, ...
    claim_holder TEXT,                                -- format: "{host}#{pid}" or "ssh:{host}#{pid}"
    claim_until  INTEGER,                             -- UNIX ms; claim auto-expires at this time
    priority     INTEGER,                             -- 1 (urgent) .. 4 (low); NULL sorts last
    tracker_kind TEXT,                                -- matches tracker registry
    tracker_id   TEXT,                                -- external ID in the tracker
    title        TEXT NOT NULL,
    description  TEXT,
    worker_host  TEXT,                                -- "local" | SSH host name; current/last worker
    workspace    TEXT,                                -- path of latest workspace (current attempt)
    archived_at  INTEGER,                             -- soft-delete; non-NULL = archived/forgotten
    created_at   INTEGER NOT NULL,
    updated_at   INTEGER NOT NULL
);

-- Hierarchy depth is enforced in code (the harness rejects parent_id pointing to a task).
-- It would also be enforceable via a recursive trigger, but that adds write-path overhead
-- for a constraint that the planning layer already validates.

CREATE TABLE phase_transitions (
    id           TEXT PRIMARY KEY,
    unit_id      TEXT NOT NULL REFERENCES units(id),
    from_phase   TEXT NOT NULL,
    to_phase     TEXT NOT NULL,
    reason       TEXT,
    transitioned_at INTEGER NOT NULL
);

CREATE TABLE task_blockers (
    task_id      TEXT NOT NULL REFERENCES units(id) ON DELETE CASCADE,
    blocked_by   TEXT NOT NULL REFERENCES units(id) ON DELETE CASCADE,
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
    id           TEXT PRIMARY KEY,                        -- ULID
    session_id   TEXT NOT NULL REFERENCES sessions(id),
    event        TEXT NOT NULL,                           -- GateBlocked | MergeConflict | Paused | UATPending
    unit_id      TEXT,
    detail       TEXT,
    created_at   INTEGER NOT NULL,
    resolved_at  INTEGER,                                 -- non-NULL = resolved; see resolution rules below
    resolved_by  TEXT                                     -- "user" | "auto" | command name (e.g. "/sf uat-approve")
);

-- Resolution rules:
--   GateBlocked    : resolved when the gate passes on a subsequent attempt OR the unit
--                    transitions to PhaseReassess; resolved_by = "auto" | "/sf force-clear"
--   MergeConflict  : resolved on /sf revert, /sf merge-resolve, or git service hook;
--                    resolved_by = command name
--   Paused         : resolved on /sf resume; resolved_by = "user"
--   UATPending     : resolved on /sf uat-approve or /sf uat-reject; resolved_by = command name
--
-- An unresolved blocker MUST be displayed in /sf status. The TUI also subscribes to
-- the corresponding pubsub event (§ 10.1) for live updates.

CREATE TABLE benchmark_results (
    id           TEXT PRIMARY KEY,
    model        TEXT NOT NULL,
    tier         TEXT NOT NULL,
    fingerprint  TEXT NOT NULL,          -- phase+complexity+project hash
    quality      REAL NOT NULL,          -- 0.0 .. 1.0
    latency_p50  INTEGER NOT NULL,       -- milliseconds
    cost_per_1k_micro_usd INTEGER NOT NULL,    -- micro-USD per 1k tokens
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

CREATE TABLE runs (
    id           TEXT PRIMARY KEY,            -- ULID
    run_kind     TEXT NOT NULL CHECK (run_kind IN ('unit_attempt', 'agent_run')),
    unit_id      TEXT REFERENCES units(id)  ON DELETE SET NULL,   -- preserve forensics
    agent_id     TEXT REFERENCES agents(id) ON DELETE SET NULL,   -- preserve forensics
    unit_id_snap TEXT,                                             -- ID at run start; survives delete
    agent_name_snap TEXT,                                          -- name at run start; survives delete
    attempt      INTEGER,                     -- only for unit_attempt
    worker_host  TEXT,
    workspace    TEXT,                        -- workspace AT THIS attempt; authoritative for this run
    started_at   INTEGER NOT NULL,
    ended_at     INTEGER,
    outcome      TEXT CHECK (outcome IS NULL OR outcome IN
                  ('success','failure','abandoned','canceled','interrupted',
                   'unit_timeout','turn_timeout','stalled')),
    error_code   TEXT,                        -- typed error from § 20.1; stores the string
                                              --   value of the const, e.g. "turn_timeout"
    input_tokens   INTEGER NOT NULL DEFAULT 0,
    output_tokens  INTEGER NOT NULL DEFAULT 0,
    cost_micro_usd INTEGER NOT NULL DEFAULT 0, -- cost in micro-USD (1e-6 USD); avoids float drift
    CHECK (
      (run_kind = 'unit_attempt'  AND unit_id_snap  IS NOT NULL AND agent_name_snap IS NULL AND attempt IS NOT NULL)
      OR
      (run_kind = 'agent_run'     AND agent_name_snap IS NOT NULL AND unit_id_snap IS NULL AND attempt IS NULL)
    )
);

-- Aggregate token/cost columns are an end-of-run rollup written once on ended_at.
-- Span data in trace.jsonl (§ 19.3) is authoritative; runs columns are the cached
-- summary used by /sf session-report and the HTTP API without re-scanning JSONL.
--
-- Soft-delete model: units and agents are NEVER hard-deleted by the harness — only
-- marked archived (units.archived_at, agents.archived_at). The snap_ columns ensure
-- run history survives even if a future operator manually drops rows.

-- Local mirror of selected Singularity Memory entries that the harness needs offline.
-- Limited to anti-patterns by default — small, high-value, MUST surface even
-- if Singularity Memory is unreachable.
CREATE TABLE local_anti_patterns (
    id           TEXT PRIMARY KEY,
    description  TEXT NOT NULL,
    context      TEXT NOT NULL,
    correct_path TEXT NOT NULL,
    source_unit  TEXT,
    fingerprint  TEXT,                        -- phase + project hash, for fast filter
    created_at   INTEGER NOT NULL,
    synced_at    INTEGER                      -- last time confirmed against Singularity Memory
);
```

### 3.2 Persistent agent tables

```sql
CREATE TABLE agents (
    id           TEXT PRIMARY KEY,
    name         TEXT NOT NULL UNIQUE,
    system       TEXT NOT NULL,          -- system prompt template
    model        TEXT NOT NULL,
    state        TEXT NOT NULL DEFAULT 'idle' CHECK (state IN ('idle','running','waiting','stopped')),
    capabilities TEXT,                   -- JSON array of capability tags; cached in agent_capabilities
    max_turns_per_run INTEGER NOT NULL DEFAULT 100,
    archived_at  INTEGER,                -- soft-delete; non-NULL = archived
    created_at   INTEGER NOT NULL,
    last_active  INTEGER
);

-- Indexed lookup table for capability matching (handoff "capability:tag1,tag2").
-- Maintained in sync with agents.capabilities by the agent CRUD layer.
CREATE TABLE agent_capabilities (
    agent_id     TEXT NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    capability   TEXT NOT NULL,
    PRIMARY KEY (agent_id, capability)
);
CREATE INDEX agent_capabilities_by_tag ON agent_capabilities(capability, agent_id);

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

**Non-active** = anything except `active`. A unit transitioning from `active` to `done` or `cancelled` mid-run triggers `ReconciliationCancel` (§ 9.2) and the attempt enters `AttemptCanceled`.

**`unknown` mid-run is a transient signal and MUST NOT cancel an in-flight attempt.** It only suppresses *new* dispatches. The orchestrator continues monitoring; if a subsequent fetch resolves the status to `done`/`cancelled`, then ReconciliationCancel fires. This protects against flaky tracker APIs.

**`blocked` mid-run** is similarly non-cancelling — the in-flight attempt continues. The blocker resurgence is logged.

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

`TrackerUnit.BlockedBy` returns upstream tracker IDs. The harness translates them to local `units.id` values when inserting/updating `task_blockers`:

1. Look up local `units.id` by `(tracker_kind, blocked_by_tracker_id)`.
2. If the local row does not yet exist, insert a placeholder unit with `phase = 'pending'` and `phase_status = 'pending'`. The next tracker fetch will fill in title/description.
3. Insert `task_blockers (task_id = local_id, blocked_by = upstream_local_id)`.

Until the upstream is materialised locally, the blocked unit remains queued — the placeholder is non-terminal and blocks dispatch by definition.

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
    PhaseUAT                   // human acceptance; only when workflow has require_uat = true
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
require_uat     = false   # if true, PhaseUAT is inserted before PhaseComplete
max_retries     = 3       # per gate in PhaseVerify
max_reassess    = 2

# .sf/workflows/release.toml — uses UAT
name            = "release"
phases          = ["research", "plan", "execute", "tdd", "verify", "review", "uat", "merge", "complete"]
require_tdd     = true
require_review  = true
require_uat     = true    # halts after UAT enters; only resumes on /sf uat-approve
max_retries     = 3

# .sf/workflows/spike.toml
name            = "spike"
phases          = ["research", "plan", "execute", "complete"]
require_tdd     = false
require_review  = false
max_retries     = 0
```

PhaseUAT halts the auto-loop with `SignalPause` and waits for `/sf uat-approve <unit-id>` (advance to PhaseMerge) or `/sf uat-reject <unit-id> "reason"` (advance to PhaseReassess). The harness MUST fail startup if a configured workflow template references an unknown phase or includes `uat` without `require_uat = true`.

#### Workflow selection at dispatch

The workflow used for a given unit is determined in this order:

1. Explicit unit metadata: tracker label `sf:workflow=<name>` (if set).
2. Project default: `[harness] default_workflow = "feature"` in `.sf/config.toml`.
3. Built-in fallback: `feature` (if available) else the first workflow in `.sf/workflows/`.

The selected workflow is recorded in `units.workflow` at dispatch time and never re-evaluated for that unit, even on retry — workflow stability across attempts is a hard guarantee. Additionally, the *content* of the chosen template is hashed (SHA-256) and stored in `units.workflow_hash`. If the on-disk template changes mid-session, the harness uses the pinned hash's content (cached in SQLite at `workflow_pins.content`) for that unit; new units pick up the new content. This prevents in-flight units from silently changing rules.

```sql
CREATE TABLE workflow_pins (
    hash         TEXT PRIMARY KEY,    -- SHA-256 of template content
    name         TEXT NOT NULL,
    content      TEXT NOT NULL,       -- frozen TOML at first pin
    pinned_at    INTEGER NOT NULL
);
```

### 4.6 PhaseReassess

`PhaseReassess` is entered when a unit cannot make progress through normal phases (gate failed `max_retries` times, merge conflict, supervisor halt). The Reassess agent is dispatched at the **`reasoning`** tier with `Think: true` and is given:

- The original task description.
- The full failure trail: gate output, last `max_retries` attempt errors, last commit history.
- The unit's plan (from `.sf/active/{unit-id}/plan.md`).

The Reassess agent MUST output one of:

| Outcome | Effect |
|---|---|
| **Re-plan** | Writes a new `plan.md`, transitions back to `PhasePlan`. Counter `max_reassess` decrements. |
| **Abandon** | Writes a `decision.md` explaining why the task cannot succeed; transitions to `PhaseComplete` with verdict `abandoned`. The tracker is updated with the abandonment reason. |
| **Escalate** | Halts auto-loop with `SignalPause`; writes a `human-question.md` with concrete questions for the operator. Resumes on `/sf reassess-resolve <unit-id>`. |

If `max_reassess` hits zero on a Re-plan path, the next entry into PhaseReassess MUST be Abandon or Escalate; Re-plan is rejected.

### 4.7 Phase transition rules

1. All phase transitions MUST go through a single `Harness.Transition(ctx, from, to, reason)` method.
2. `Transition` MUST persist the `PhaseTransition` record to SQLite BEFORE the new phase begins. A crash mid-phase means on resume the harness re-enters the last committed phase cleanly (see § 4.8).
3. `Transition` MUST emit a pubsub `PhaseChange` event after the SQLite write. The TUI subscribes — it MUST NOT poll phase state directly.
4. The harness MUST set `Think: true` on the model config for `Research`, `Plan`, and `Reassess` phases. The agent does not control this.
5. **`PhaseChange` is non-vetoable.** Hook subscribers receive a notification *after* the transition is committed; they cannot block or reject. Hooks that need veto semantics MUST register on `PreDispatch` instead, which fires before the next dispatch and IS vetoable.

### 4.8 Crash recovery

In-memory scheduler state is intentionally not persisted (§ 20.2). On restart, the orchestrator MUST follow this exact sequence:

1. **Acquire project lock** at `<project>/.sf/run.lock` (PID file). Stale lock (PID not in `/proc` on Linux, `kill(pid, 0)` on other Unixes) is cleaned and logged. The lock is per-project; multiple projects can run auto concurrently on the same machine.
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

### 5.3.1 Atomic claim acquisition

The orchestrator acquires a claim with a single conditional UPDATE:

```sql
UPDATE units
   SET claim_holder = ?, claim_until = ?, phase_status = 'running', updated_at = ?
 WHERE id = ?
   AND (claim_holder IS NULL OR claim_until < ?);  -- ? = now()
```

Dispatch proceeds only if `rows_affected = 1`. This makes the claim race-free at the DB level and supports multiple orchestrators against the same `~/.sf/sf.db` even though SF normally runs as a singleton (one process per `~/.sf/run.lock`). The atomic claim is the safety net if the lock fails (e.g. shared NFS, broken filesystem semantics).

`units.attempt` is the **current** attempt counter (used as the `attempt` prompt template variable). Historical attempts live in `runs` (§ 3.1). Authority: `units.attempt` is incremented exactly when a new `runs` row is inserted; the two are kept in sync inside the same transaction.

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

### 5.4.1 Turn outcome signal

Between transport-level "turn ran cleanly" and phase-level "gate passed," the harness MUST capture a per-turn semantic signal. After every turn, the harness inspects the model output for an explicit terminal marker:

| Marker (in agent output) | Meaning | Effect |
|---|---|---|
| `<turn_status>complete</turn_status>` | Agent considers this turn's goal achieved | Recorded; allow continuation if max_turns_per_attempt not reached |
| `<turn_status>blocked</turn_status>` | Agent stuck, need user input or escalation | Triggers `SignalPause` if auto-mode |
| `<turn_status>giving_up</turn_status>` | Agent has decided the task can't be done | Ends attempt; transitions to PhaseReassess |
| (no marker) | Default success | Continue normally |

The marker is parsed from the last 200 chars of the agent's response. Markers appearing earlier are ignored (prevents partial-quote false positives). This gives the harness a checkpoint *between* turns without waiting for a phase boundary.

The agent prompt template (`prompts/execute-task.md`) instructs the agent to emit one of these markers at end-of-turn. Compliance is best-effort — absence of a marker is treated as default success.

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

The `attempt` variable enables prompt templates to give different instructions to retrying agents vs. fresh starts. A retry prompt SHOULD include: `"your previous attempt failed with: {{last_error}} — focus on that specifically."` The harness injects `last_error` automatically on `attempt >= 2`.

**`last_error` is only injected on `TurnFirst` of attempts ≥ 2.** Continuation turns within the same attempt have already established context and don't need it. A turn failure within an attempt always fails the entire attempt (§ 6); there are no mid-attempt error injections to reason about.

`last_error` content MUST be capped at 4 KB. Larger payloads (gate output, lint dumps, traceback) are truncated head-and-tail: 2 KB from the start, marker `... [truncated, full payload at <path>] ...`, then 2 KB from the end. The full payload is written to `.sf/active/{unit-id}/last-error-full.txt` so the agent can `read_file` it if the truncated context isn't enough.

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

1. Write a `session_summary` entry to Singularity Memory via `retain`.
2. Clear the hot cache (in-memory last-N turns).
3. Start the next turn with a fresh context window seeded by a `recall` from Singularity Memory.

Compaction MUST NOT truncate the window — it MUST replace it with a fresh recall. A truncated window loses structure; a recalled window gains relevance.

**Agent run compaction preserves the wake context.** For persistent agent runs, the compacted window MUST include verbatim:
- The wake message that started this run.
- The most recent 3 inbox arrivals delivered in this run.
- The agent's full `agent_memory_blocks` (these are durable anyway, but they go above the recall block).

Compaction without this preservation can drop the originating intent and cause the agent to lose thread continuity mid-run.

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

### 9.5 SignalAbort and in-flight tool calls

When the harness receives `SignalAbort` while a tool call is in flight (e.g. a long-running `bash` subprocess), it MUST follow this sequence:

1. Cancel the tool call's context (Go `context.CancelFunc`). Cooperative cancellation MUST be honoured by built-in tools.
2. Wait up to `[harness] tool_abort_grace = "5s"` for the tool to exit cleanly.
3. After the grace period, send `SIGTERM` to any tool subprocess.
4. Wait an additional `[harness] tool_abort_kill = "3s"`.
5. If the subprocess is still running, send `SIGKILL`.

Total worst case: 8 seconds from `SignalAbort` to forcible termination. The harness MUST NOT hang the orchestrator waiting on a non-cooperating tool call.

After the tool call ends (cleanly or via SIGKILL), the harness records the run as `outcome = canceled` with `error_code = canceled_by_supervisor` and emits the `after_run` hook before releasing the slot.

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
- Hook timeouts are per-hook-type. Defaults:

  | Hook | Default | Rationale |
  |---|---|---|
  | `before_run` | `120s` | Cleanup, dependency install can take time |
  | `after_run` | `30s`  | Best-effort teardown |
  | `after_create` | `120s` | First-time setup |
  | `before_remove` | `30s` | Cleanup |
  | `pre_dispatch` | `15s` | Should be a fast check |
  | `post_unit` | `60s` | Subprocess work; longer for git push |
  | `doc_sync` (built-in) | `5m` | Runs an agent dispatch over the diff |

  All overridable in config via `[harness.hooks.timeouts.<hook_name>] = "<duration>"`. A timeout kills the hook and logs. A `PostUnit` hook timeout MUST NOT block the next dispatch.
- The git service subscribes to PostUnit via a hook and handles commits, branch creation, and push. The harness MUST NOT call `git` directly.
- Singularity Memory feedback (retain learnings, mark anti-patterns) is emitted from a built-in PostUnit hook (not a subprocess) — it calls the Singularity Memory client directly.
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

### 10.5.0 SF tool registration

Crush uses `internal/agent/tools/` to register tool implementations exposed to the agent. SF adds new tools (`send_message`, `core_memory_append/replace`, `handoff`, `wait_for_reply`, `chapter_open`, `stop`, `tracker_query`, etc.) by registering them at boot via the same Crush API. There is NO parallel tool registry — SF tools live in `harness/tools/` and call into Crush's registration during `init()`.

SF-specific tools MUST:
1. Conform to the response shape of § 10.4 (`{success, output, contentItems}`).
2. Honour Crush's `PreToolUse` hook system — they receive the same hook pipeline as built-in tools.
3. Document the auto_approve key they expect (e.g. `agent:send_message`) so projects can list them in `[harness.auto_approve.tools]`.

This means PreToolUse hooks can deny SF tool calls just like any other; the auto-approve list scopes them; permissions are uniform.

### 10.5 Doc sync (sub-step of PhaseMerge or PhaseComplete)

Doc sync runs as the final sub-step of the **last code-mutating phase** before `PhaseComplete`:

- For workflows that include `PhaseMerge`: doc sync runs at end of `PhaseMerge`.
- For workflows that omit `PhaseMerge` but include `PhaseExecute` (e.g. `spike`): doc sync runs at end of the last code-mutating phase that ran. If the spike adopted a new dependency, doc sync still gets a chance to update `STACK.md`.

It is not a separate phase and not a post-merge hook; it is the final sub-step of whichever phase was last to mutate code.

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

### 12.3 Merge ordering for parallel slices

When multiple slices in `branch-per-slice` mode complete concurrently, the harness MUST merge them in **dependency-aware** order, not completion order:

1. A slice marked `code_depends_on: ["m1/s2"]` in unit metadata is held until that upstream slice's branch has merged.
2. With no declared code dependency, slices merge in `created_at` order.
3. The merge gate is serial: only one slice's merge runs at a time per project, even if multiple are eligible.

This is distinct from `task_blockers` (task-completion dependency). **Code dependency** means slice B's diff cannot merge cleanly before slice A's diff. Without explicit declaration, the harness assumes no code dependency and merges in creation order — accept that this can produce avoidable conflicts that the next attempt will resolve.

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
- Fail increments the gate-level retry counter (separate from `units.attempt`). The gate retry counter resets on the next phase transition.
- Default max gate retries: 3. Configurable per gate via `[harness.gates.max_retries.<gate-name>]`.
- On retry, the harness re-dispatches the same unit with gate failure output appended to context. The agent MUST see what failed and why.
- After max retries, the harness transitions to `PhaseReassess` and emits `GateBlocked` on pubsub.
- Gate results MUST be stored in `gate_results` table and written as span events on the unit span.

### 13.2.1 Gate script protocol

Every gate script MUST adhere to this contract. Implementations that violate any rule are rejected at startup validation.

**Environment variables provided:**

| Variable | Value |
|---|---|
| `SF_PROJECT_ROOT` | Absolute path to project root |
| `SF_HOME` | SF data directory (`~/.sf` or override) |
| `SF_UNIT_ID` | Active unit ID (§ 2 format) |
| `SF_RUN_ID` | Active run ULID |
| `SF_PHASE` | Phase name (e.g. `verify`) |
| `SF_ATTEMPT` | Attempt counter, 1-indexed |
| `SF_GATE_NAME` | This gate's name (script basename without extension) |
| `SF_GATE_RETRY` | Gate retry counter, 0-indexed |
| `SF_WORKSPACE` | Path of the unit's workspace |
| `SF_TRACE_FILE` | Path to current day's trace JSONL |

**Stdin:** the `UnitResult` JSON struct (§ 10.2). UTF-8, single line, terminated with `\n`.

**Exit code:** `0` = pass; `1` = fail (retry); `2` = block (do not retry, transition straight to PhaseReassess); `3` = skip (gate is not applicable for this unit). Other codes are treated as `1`.

**Stdout / stderr:** captured combined, truncated at 8 KB, stored in `gate_results.output`. Multi-line is fine. No structured output is required, but if the first line is valid JSON of the form `{"summary": "...", "issues": [...]}` the harness uses it for richer reporting.

**Timeout:** default 5 minutes per gate, configurable via `[harness.gates.timeouts.<gate-name>]`. Timeout = SIGTERM, then 10s grace, then SIGKILL; recorded as `error_code = "gate_timeout"`.

**Cwd:** the workspace directory. Scripts MAY assume `git status` etc. work as expected.

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

Large diffs MUST NOT be reviewed in a single pass. The harness MUST split the changed file list into chunks of ≤ 300 lines (`ReviewChunkLines = 300`) before dispatching the review agent. Files larger than `ReviewChunkLines` get their own chunk.

To prevent context-blind review of cross-file changes, the harness runs three passes:

1. **Establish-context pass (single dispatch, fast tier).** The agent receives the full diff summary (file list + first/last 20 lines of each) and produces a one-paragraph "what this change does and what to watch for" summary.
2. **Per-chunk review pass (parallel, `standard` tier).** Each chunk receives: the establish-context summary as a system-prompt prefix, then its own files. Reviewer findings are accumulated. Parallelism is bounded by `max_agents_by_phase.review`.
3. **Synthesis pass (single dispatch, `standard` tier).** All chunk findings are merged, deduplicated, and prioritised. The synthesis agent decides whether the review should pass, request changes, or block (security/correctness issue).

The synthesis verdict is what the harness acts on — chunked passes alone never decide.

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
unit_timeout          = "10m"   # default per-attempt cap; can override per phase
turn_timeout          = "5m"    # bounds one model turn
stall_timeout         = "2m"    # AttemptStalled when no agent event for this long
tool_abort_grace      = "5s"    # cooperative cancel window before SIGTERM
tool_abort_kill       = "3s"    # SIGTERM-to-SIGKILL window
max_turns_per_attempt = 50
max_attempts          = 6       # exponential backoff before giving up
hot_cache_turns       = 10      # in-memory recent-turn buffer
supervisor_interval   = "10s"
max_retry_backoff     = "5m"
doc_sync              = true
turn_input_required   = "soft"  # or "hard"
worktree_mode         = "branch-per-slice"

[harness.unit_timeout_by_phase]
research = "30m"   # AST analysis / spec reading can take real time
plan     = "20m"
execute  = "15m"
tdd      = "10m"
verify   = "10m"
review   = "15m"
merge    = "5m"
reassess = "20m"
uat      = "0"     # 0 = no timeout (UAT can take days; advance via /sf uat-approve)

[harness.concurrency.max_agents_by_phase]
execute  = 4
tdd      = 4
verify   = 10      # mostly reads — cheap
review   = 4       # parallel chunked review (§ 13.3)
merge    = 1       # serial per project (§ 12.3)

[harness.concurrency]
max_agents = 10                  # global cap; per-phase caps under [harness.concurrency.max_agents_by_phase] above

[harness.auto_approve]
tools = ["bash:read", "fs:read", "git:status", "git:diff"]

[harness.hooks]
pre_dispatch = ["./hooks/pre-dispatch.sh"]
post_unit    = ["./hooks/post-unit.sh"]
after_create = "./hooks/after-create.sh"
before_run   = "./hooks/before-run.sh"
after_run    = "./hooks/after-run.sh"
before_remove = "./hooks/before-remove.sh"

[harness.hooks.timeouts]   # per-hook overrides; defaults in § 10.3
before_run = "120s"
post_unit  = "60s"
doc_sync   = "5m"

[providers]
# Crush-inherited provider settings live here.
# API keys MUST use vault:// (§ 24); plaintext is rejected at startup.
anthropic.api_key = "vault://secret/singularity-crush#anthropic_api_key"
openai.api_key    = "vault://secret/singularity-crush#openai_api_key"

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

[memory]
mode    = "embedded"                              # "embedded" (default) | "remote"
url     = "http://memory.tailnet.local:7843"     # required when mode = "remote"
api_key = "vault://secret/singularity-crush#sm_api_key"  # required when mode = "remote"
# Embedded mode runs the singularity_memory_server engine in-process.
# Remote mode shares the server across the fleet (Hermes, OpenClaw, sf, etc.).

[worker]
ssh_hosts                      = []
max_concurrent_agents_per_host = 3
ssh_auth_method                = "agent"   # "agent" | "key" | "key+agent"
ssh_identity_file              = "~/.ssh/id_ed25519"  # used for "key" or "key+agent"
ssh_known_hosts                = "~/.ssh/known_hosts" # MUST verify; no auto-trust
ssh_disconnect_timeout         = "30s"
host_quarantine                = "5m"

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

Changing session-immutable fields requires restart. **If a dynamic reload detects a changed session-immutable field, the harness MUST**:

1. Log a warning naming the field, old value, new value.
2. Continue using the in-process value for the current session.
3. Display the change in `/sf status` as "config drift detected — restart to apply: <fields>".
4. NOT crash and NOT auto-restart.

### 14.4 Startup validation

The harness MUST validate config at startup and MUST fail fast with a descriptive error on invalid config. It MUST NOT silently ignore unknown keys or bad values. `/sf doctor` MUST run `HarnessConfig.Validate()` as one of its checks.

### 14.5 Plan.md format

Every active unit has a `.sf/active/{unit-id}/plan.md` written by `PhasePlan` and consumed by all subsequent phases. The format is:

```markdown
---
unit_id: task/m1/s2/t3
created_at: 2026-04-29T14:22:00Z
written_by: claude-sonnet-4-6
plan_version: 1
---

# Goal

<one paragraph: what success looks like>

# Approach

<2-3 paragraphs: how the agent intends to do it>

# Deliverables

- [ ] <concrete file or behavioural change>
- [ ] <…>

# Verification

- <gate or check that proves done>
- <…>

# Notes

<context, gotchas, anti-patterns to avoid for this unit>
```

The frontmatter `plan_version` increments on each PhaseReassess→Re-plan. Subsequent phases parse the frontmatter to detect plan version changes (informational; not load-bearing).

The harness MUST validate that `plan.md` parses as Markdown with the required frontmatter fields before allowing a transition out of `PhasePlan`. Missing `# Goal` or `# Deliverables` sections fail the phase.

### 14.6 Project directory layout

Every project has a `.sf/` directory with this canonical layout:

```
<project>/
├── .sf/
│   ├── config.toml             # project config (§ 14.1)
│   ├── workflows/              # workflow templates (§ 4.5)
│   │   ├── feature.toml
│   │   └── spike.toml
│   ├── hooks/                  # hook scripts referenced by config
│   ├── gates/                  # gate scripts referenced by config
│   ├── sf.db                   # SQLite orchestration DB
│   ├── run.lock                # process lock (§ 4.7)
│   ├── auto.lock               # signals auto-mode active (§ 4.7)
│   ├── active/                 # in-progress unit artifacts
│   │   └── {unit-id}/          # one directory per active unit
│   │       ├── plan.md         # unit's plan/notes
│   │       ├── workspace -> /path  # symlink to actual workspace
│   │       └── run-{run-id}.log    # per-run log
│   ├── archive/                # completed work + age-rolled artifacts
│   │   ├── {YYYY-MM-DD}-{unit-id}/  # one per completed unit
│   │   ├── agents/             # rolled agent_inbox/messages
│   │   └── lost-learnings.jsonl    # pending_retain ages out here (§ 16.1)
│   ├── log/
│   │   └── sf.log              # rolling structured log (§ 19.2)
│   ├── runtime/
│   │   ├── paused-session.json # written when SessionPaused
│   │   ├── gate-state.json     # last gate result per unit
│   │   └── server.port         # actual HTTP API port (§ 14.2)
│   └── trace/
│       ├── trace-{YYYY-MM-DD}.jsonl  # daily-rotated spans
│       └── _meta.json          # trace schema version, file index
```

Layout is stable: `/sf revert`, `/sf history`, archive sweeps, and the HTTP API all assume these exact paths.

---

## 15. Model Routing

### 15.1 Three tiers

The tier names are fixed: `fast`, `standard`, `reasoning`. Custom tier names are NOT supported — adding a tier would force changes in routing config, complexity-upgrade logic, and the rate-feedback fingerprint, with little benefit. Each tier holds multiple candidate models in `[tiers.<name>]`. The router picks within the tier; it does not change the tier assignment.

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

The knowledge layer is **Singularity Memory** (`sm`) — an HTTP + MCP server we own. The engine was derived from [`vectorize-io/hindsight`](https://github.com/vectorize-io/hindsight) (MIT) and assimilated into `singularity_memory_server/` under our namespace; from sf's perspective there is no upstream service. The same `sm` server is shared across our agent fleet (Hermes, OpenClaw, Claude Code, Cursor, sf), so memories accumulate across tools.

singularity-crush uses `singularity-memory-client-go`, generated from `singularity-memory/openapi.yaml`. There is no local vector store, no sqlite-vec table, no FTS5 fallback — all retrieval and persistence go through `sm`.

**Embedded vs remote deployment.** sm supports both modes:

| Mode | When | Config |
|---|---|---|
| **Embedded** (default for single-user sf) | sm engine runs in-process; no extra service to operate | `[memory] mode = "embedded"` |
| **Remote** | sm runs as a tailnet service shared across multiple tools/users | `[memory] mode = "remote"`, `[memory] url = "http://memory.tailnet.local:7843"` |

Embedded mode eliminates the network hop for the common case. Switching to remote shares context across the fleet at the cost of a network round-trip per recall.

SQLite in singularity-crush holds **orchestration state only** (sessions, units, blockers, gates, benchmarks, circuit breakers, agents). Memories, learnings, anti-patterns, and codebase context live in Singularity Memory.

When `sm` is unreachable, the harness MUST log a warning and dispatch with no recall context (plus the local `local_anti_patterns` mirror, § 3.1). The agent still runs; it just lacks historical memory for that session. The harness MUST NOT block dispatch on memory availability.

**Retain failures queue locally.** PostUnit retain calls that fail (transport error, 5xx) MUST be enqueued in `pending_retain` and retried with exponential backoff on every poll tick until success. This means a unit's learnings are never silently lost to an `sm` outage:

```sql
CREATE TABLE pending_retain (
    id           TEXT PRIMARY KEY,         -- ULID
    bank         TEXT NOT NULL,
    payload      TEXT NOT NULL,            -- serialised retain request
    attempts     INTEGER NOT NULL DEFAULT 0,
    next_retry_at INTEGER NOT NULL,
    last_error   TEXT,
    created_at   INTEGER NOT NULL
);
```

`pending_retain` rows older than 7 days are flushed to `.sf/archive/lost-learnings.jsonl` and removed; at that point the operator is expected to investigate.

### 16.1.1 Memory client interface

The harness uses `singularity-memory-client-go` (auto-generated from `openapi.yaml`) through a thin wrapper that the rest of the codebase depends on. This wrapper is the seam between sf and Singularity Memory; tests substitute a fake.

```go
type Memory interface {
    // Recall fetches top-k entries from a bank for a query. opts.Filter
    // may include {"collection": "anti_patterns"} or other tags.
    Recall(ctx context.Context, bank string, query string, opts RecallOpts) ([]Entry, error)

    // Retain stores a new entry in a bank. document_id is required for
    // upsert-by-content-hash semantics (§ 16.3).
    Retain(ctx context.Context, bank string, entry Entry) error

    // Feedback signals helpfulness of an entry recalled in this dispatch.
    // signal ∈ {-1, 0, +1}; +1 resets decay timer.
    Feedback(ctx context.Context, entryID string, signal int) error

    // Validate marks the entry as still-relevant (resets decay timer).
    // Called by PostUnit when a recalled entry directly contributed to success.
    Validate(ctx context.Context, entryID string) error

    // Health probe. Used by /sf doctor and the retain queue.
    Health(ctx context.Context) error
}

type RecallOpts struct {
    TopK          int
    Filter        map[string]string
    RerankQuality string                // "fast" | "accurate"
}

type Entry struct {
    DocumentID string                   // content hash; upsert key
    Content    string
    Tags       []string
    Metadata   map[string]string        // includes maturity, decay_factor, etc.
    Score      float64                  // populated on Recall, ignored on Retain
}
```

The wrapper is responsible for:
1. Translating sf's `last_error` and gate output into `Entry.Content`.
2. Adding `is_negative` and `collection` tags appropriately.
3. Routing transport errors through `pending_retain` (§ 16.1).
4. Exposing the local `local_anti_patterns` mirror to `Recall` when `sm` is unreachable.

### 16.2 Memory tiers

Two tiers prevent token bloat during long-running sessions:

**Hot cache** — current dispatch's recent turns held in memory (never persisted to SQLite). Configurable size: `[harness] hot_cache_turns = 10`. Cleared on compaction.

**Singularity Memory store** — durable. PostUnit writes summaries, learnings, and anti-patterns. Pre-dispatch reads top-N most relevant entries. On compaction, the hot cache is summarised and written to Singularity Memory as a `session_summary` entry.

The harness MUST NOT mix the two tiers.

### 16.3 Two-bank pattern

Each session uses two Singularity Memory banks, queried separately and merged before each dispatch:

```go
projectRecall := sm.Recall("project/"+projectHash, query)
globalRecall  := sm.Recall("global/coding", query)
// merge, deduplicate, inject top-N into unit context
```

`projectHash` is derived deterministically (so the same project hits the same bank from any machine):

1. If the project root is a git repository, `projectHash = sha256(canonical_remote_url)[:16]` where canonical_remote_url is the `origin` URL normalised (strip auth, lowercase host, drop trailing `.git`).
2. If no git remote, `projectHash = sha256(absolute_path_with_real_user_home)[:16]`.
3. The resolved hash is cached in `.sf/runtime/project-hash.json` to ensure stability if the remote changes (a cleared cache forces re-derivation; a project move under a different remote is a deliberate re-bank).

This means a developer cloning the repo on a second machine hits the same Singularity Memory bank as their first machine. Different forks of the same project have different remotes and thus different banks — desired, because their context diverges.

Concurrent `retain` calls from parallel slice workers use `document_id` derived from content hash. Duplicate memories silently overwrite rather than accumulate.

### 16.4 Anti-pattern library

Anti-patterns are memories tagged `collection: anti_patterns`, `is_negative: true`. They:
- Are written explicitly when the agent makes a mistake (gate failure or user feedback).
- MUST NOT be subject to normal maturation decay — they persist at full weight until explicitly removed.
- Are retrieved at dispatch time and presented in a dedicated block: `<anti_patterns>avoid these mistakes...</anti_patterns>`.
- MUST also be mirrored to the local `local_anti_patterns` SQLite table (§ 3.1) on `retain`. When Singularity Memory is unreachable, the harness still injects local anti-patterns into prompt context. Anti-patterns are small, high-value, and never decay — making them the one knowledge category worth duplicating locally.

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

Retrieval is delegated to Singularity Memory via `sm.Recall(bank, query, opts)`. Singularity Memory runs its own internal pipeline — fused semantic + lexical retrieval, optional reranking, and decay weighting — and returns ranked entries. The harness does not implement a retrieval pipeline of its own.

Recall options the harness uses:

| Option | Use |
|---|---|
| `top_k` | Number of entries to inject into prompt (default 5) |
| `bank` | `project/{hash}` or `global/coding` (§ 16.3) |
| `filter` | Tag filters (e.g. `collection=anti_patterns`) |
| `rerank_quality` | `fast` (routine) or `accurate` (pre-dispatch context injection) |

The harness applies its own maturity and anti-pattern weighting (§ 16.4, § 16.5) by tagging entries on retain and filtering / re-ordering on recall — Singularity Memory stores the metadata but does not interpret it.

### 16.8 `sf init`

Deep analysis is default, not opt-in:

1. AST-level codebase scan (languages, structure, entry points, dependencies).
2. Git history analysis (active areas, recent changes, contributors).
3. Retain findings into the `project/{hash}` Singularity Memory bank.
4. Establish `.sf/config.toml` with detected stack, workflow templates, model routing hints.

`--quick` flag skips Singularity Memory indexing for throwaway sessions.

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

### 17.5 Agent run termination

A persistent agent run terminates when ANY of:

1. **Inbox drained.** The agent's inbox has no `delivered = 0` rows AND the agent's last turn produced no outgoing `send_message` requiring `wait_for_reply`.
2. **Explicit stop.** The agent calls a built-in `stop()` tool, signalling it has no further work.
3. **Budget exhausted.** Per-agent `Budget.AtHardLimit()` fires (§ 8). Compaction does NOT terminate the run; only hard-limit does.
4. **Turn cap.** `max_turns_per_run = 100` (configurable per-agent via `agents.max_turns_per_run` column or `[harness] agent_max_turns_per_run`). Higher than unit cap because agents are long-running.
5. **Supervisor signal.** `SignalAbort` for any reason (StuckLoop, AbandonDetect, ReconciliationCancel does not apply to agents).
6. **Timeout.** A configurable `agent_run_timeout = "30m"` from run start.

On termination the agent transitions to `AgentIdle` (or `AgentStopped` for case 2). On wake (next inbox message), a NEW run begins — the **agent's hot cache is NOT preserved across runs**; only the durable memory blocks (`agent_memory_blocks`) and message history (`agent_messages`) survive.

### 17.6 Agent fleet supervision

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

`handoff(to, context)` transfers the active task to a specialist agent. `to` is either an agent name (exact match) or a capability tag string (e.g. `"capability:go"` or `"capability:sql,perf"`):

1. **Resolution.** If `to` starts with `capability:`, the harness queries `agents` for an active agent (`archived_at IS NULL`, `state != 'stopped'`) whose `capabilities` JSON array includes ALL listed tags. If multiple match, the one with the lowest `last_active` wins (round-robin). If none match, `handoff` returns `ErrNoCapableAgent`.
2. **Suspension.** The calling agent's current run is suspended (not completed).
3. **Context delivery.** The target agent receives the full task context (system prompt, memory blocks at handoff time, last N messages) pre-loaded as a snapshot in its inbox.
4. **Wait.** The calling agent transitions to `AgentWaiting` until the specialist replies (subject to `wait_for_reply` timeout).
5. **Fallback.** If the target agent is not found or is `AgentStopped`, `handoff` returns an error and the calling agent continues.

```go
// Tool the agent calls:
// handoff(to: string, context: string) -> HandoffTicket
// Agent calls wait_for_reply(ticket.id) to block until the specialist responds.
//
// to formats:
//   "go-specialist"             — exact agent name
//   "capability:go"             — first eligible agent with capability tag "go"
//   "capability:sql,perf"       — agent with both "sql" AND "perf" tags
```

Capability matching is the recommended form — it lets the agent fleet evolve without changing handoff call sites.

### 18.5 Append-only inbox log

`agent_inbox` MUST be append-only. Rows MUST NOT be deleted after insert. `delivered` is the only mutable column. This gives a complete audit trail of all inter-agent communication.

Inbox and message tables are subject to a periodic GC sweep: rows with `delivered = 1` and `created_at < now() - retain_window` are moved to `.sf/archive/agents/{agent_id}/inbox-{YYYY-MM}.jsonl` and deleted from the live tables. Default `retain_window = 30d`, configurable via `[harness] agent_inbox_retain = "30d"`. The archive is human-readable and queryable by `/sf agent history`.

### 18.6 Memory block concurrency

An agent's memory blocks are owned by that agent — they are NEVER shared with other agents (§ 18.7). Within a single agent, a turn's tool calls execute serially (one tool at a time), so two `core_memory_*` writes within a turn cannot race. Across turns, the harness commits the prior turn's writes before dispatching the next turn (§ 17.3).

`handoff` does NOT share blocks — the receiving agent gets its own blocks. The `context` argument of `handoff` is a snapshot, not a reference.

### 18.7 What not to build

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
- Spans MUST be written to `<project>/.sf/trace/trace-{YYYY-MM-DD}.jsonl` (rolls at local-midnight on first span emission after midnight).
- Span emission MUST be non-blocking — use a buffered channel with a background writer goroutine.
- MUST NOT drop spans. If the buffer is full, block briefly rather than discard.
- The first line of each daily file MUST be a `_meta` record:
  ```json
  {"_meta":true,"trace_schema_version":1,"sf_version":"<semver>","created_at":"<rfc3339>"}
  ```
  Readers branch on `trace_schema_version`. Future schema changes bump the version; no in-place migration of historical files.

### 19.3.1 Trace index for forensics

JSONL is the source of truth for spans, but `/sf forensics` queries demand fast access to specific runs/units/sessions. The harness MUST maintain a small SQL index alongside the JSONL:

```sql
CREATE TABLE trace_index (
    run_id       TEXT NOT NULL,
    span_id      TEXT NOT NULL,
    parent_span_id TEXT,
    trace_id     TEXT NOT NULL,
    operation    TEXT NOT NULL,        -- "tool_call" | "phase_transition" | "model_request" | "hook"
    started_at   INTEGER NOT NULL,
    duration_ms  INTEGER,
    file_path    TEXT NOT NULL,        -- which JSONL file holds the full record
    file_offset  INTEGER NOT NULL,     -- byte offset within the file
    PRIMARY KEY (run_id, span_id)
);
CREATE INDEX trace_index_started_at ON trace_index(started_at);
CREATE INDEX trace_index_trace_id ON trace_index(trace_id);
```

The index is populated by the trace writer goroutine after a successful flush. `/sf forensics <run-id>` queries the index, then seeks into the JSONL files for full payloads.

JSONL files older than 30 days MAY be moved to `<project>/.sf/archive/trace/` by `/sf clean`. The move MUST be a single transaction:
1. Move the JSONL file to `archive/trace/`.
2. UPDATE `trace_index SET file_path = REPLACE(file_path, '.sf/trace/', '.sf/archive/trace/') WHERE file_path = ?`.

Both steps under a process-level lock so a concurrent forensics query never observes a half-renamed state. If `/sf clean` is interrupted mid-move, on next run it detects the file in archive but index pointing to original path and repairs by re-running the UPDATE.

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
2. **Singularity Memory recall** — completed chapters are stored as discrete entries. Recall queries match against chapter intent.

The agent MAY open a chapter explicitly via `chapter_open(name)`.

### 19.5 HTTP observability API

The harness MUST expose a lightweight HTTP server on `localhost` when `server.port` is configured. The API is observability-only — orchestrator correctness MUST NOT depend on it.

**Auth.** The server binds to `127.0.0.1` only. Every request MUST include header `Authorization: Bearer <token>` where token is read from `<project>/.sf/runtime/api.token` (generated as 32 random bytes hex on first start, mode 0600). Multi-user machines need this — `localhost` alone is insufficient. The actual port and token are written to `<project>/.sf/runtime/server.port` and `api.token` for tools to discover.

**Session filter.** All endpoints accept `?session=<id>` to scope the response to one session. With no parameter, responses include all active sessions in the project DB; the response body has a top-level `sessions: [...]` array with the snapshot per session.

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

The harness MUST track the latest rate-limit payload from any provider event and surface it in the TUI and HTTP API. Rate-limit data is observability-only — no retry logic is driven by it.

**Why not actively throttle on rate limits?** Three reasons: (a) rate limit headers vary in format and meaning across providers (Anthropic's `anthropic-ratelimit-tokens-remaining` vs OpenAI's `x-ratelimit-remaining-tokens` differ in semantics — input-only vs total), (b) the model router (§ 15) already moves between providers, so a single provider's pressure does not need to feed back into dispatch, (c) the circuit breaker (§ 9.3) handles repeated provider failures including 429. Rate-limit data is for the operator to see what's happening, not for the orchestrator to react to.

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
    ErrNoCapableAgent           = "no_capable_agent"
    ErrSshDisconnected          = "ssh_disconnected"
    ErrCanceledBySupervisor     = "canceled_by_supervisor"
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

In auto-mode the harness calls Crush's existing `permission.AutoApproveSession()` ONLY for operations listed in `[harness.auto_approve]`. Sensitive operations (`fs:write-outside-project`, `shell:exec`) MUST always prompt regardless of auto-mode setting.

**Precedence between PreToolUse hooks and auto-approve.** Crush's PreToolUse hook system already runs before any tool call. If a PreToolUse hook returns `deny` or `halt`, the tool call is rejected even if `auto_approve` lists the tool. The order is:

1. PreToolUse hooks run first; their decision is final for `deny`/`halt`.
2. If hooks return `allow` or no decision, the auto-approve list is consulted.
3. If neither approves, the user is prompted (interactive mode) or the call fails (auto-mode for non-allowlisted tools).

This means: PreToolUse hooks MAY revoke an auto-approval; the auto-approve list MAY NOT override a hook denial. This precedence is critical for security policies that need to override per-session approvals.

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

### 22.3 Disconnect and zombie handling

When the SSH connection drops mid-turn:

1. The orchestrator marks the attempt `failed` with `error_code = "ssh_disconnected"` after `[worker] ssh_disconnect_timeout = "30s"` of no stdio activity.
2. **Before** scheduling a retry, the orchestrator MUST emit a remote-cleanup script over a fresh SSH session: `pgrep -f "<workspace_marker>" | xargs -r kill -TERM`, wait 10s, then `kill -KILL`. The marker is a unique string injected into the agent process's command line (e.g. `--sf-run-id=<run_id>`).
3. If the cleanup script fails (host unreachable), the host is marked `unhealthy` for `[worker] host_quarantine = "5m"`. New dispatches skip it; the host re-eligibility check runs each tick.
4. The retry MUST land on a different host if `host_quarantine` is in effect for the original host; otherwise same host with a fresh workspace re-creation (the previous workspace is moved to `~/.sf/orphaned-workspaces/{timestamp}-{run-id}/` for forensics, not deleted).

Zombies are the dominant failure mode for distributed execution; ignoring them produces double-write corruption.

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

### `/sf rate over|ok|under [unit-id]`

Signal model quality. Without `unit-id`, targets the most recently completed run in the current session — specifically the latest row in `runs` where `outcome IN ('success', 'failure')` and `ended_at IS NOT NULL`, scoped to `session_id`. With `unit-id`, targets the latest run for that unit.

Writes to `benchmark_results` with the human-rating weight multiplier (default 3×). Cannot be issued against an in-flight run.

### `/sf benchmark`

Run on-demand model benchmarks for all tiers against real task samples. Updates `benchmark_results`.

### `/sf doctor`

Run health checks:
- `HarnessConfig.Validate()`
- Vault connectivity
- Singularity Memory connectivity
- SQLite schema version
- Lock file state
- Workflow template syntax
- HTTP API token presence + permissions

Exit code: `0` if all checks pass, `1` if any FAIL or WARN. Useful in CI: `sf doctor || exit 1`. The TUI rendering shows pass/warn/fail per check; the JSON form (`/sf doctor --json`) returns a structured report for automation.

### `/sf history`

Query archived units in `.sf/archive/`. Supports filtering by date, phase, model, verdict.

### `/sf forensics`

Inspect the trace for a specific unit or session. Shows all spans, tool calls, phase transitions, and gate results in chronological order.

### `/sf reset-circuits`

Clear all tripped circuit breakers. Next dispatch uses benchmark scores to select within each tier normally.

### `/sf reassess-resolve <unit-id> "operator response"`

Resume a unit that entered `PhaseReassess` with the **Escalate** outcome (§ 4.6). The operator's response is appended as the next attempt's `last_error` so the agent can incorporate it. The unit re-enters `PhasePlan`.

### `/sf force-clear <blocker-id>`

Operator override: mark a `session_blockers` row resolved with `resolved_by = "/sf force-clear"`. Used to dismiss stuck `GateBlocked` events that can't auto-resolve (e.g. flaky external test infrastructure).

### `/sf merge-resolve <unit-id>`

Resume a unit halted on `MergeConflict`. Assumes the operator has resolved the conflict in the worktree. Triggers re-emission of `MergeReady`.

### `/sf uat-approve <unit-id>` and `/sf uat-reject <unit-id> "reason"`

Advance a unit out of `PhaseUAT` (§ 4.6). Approve transitions to `PhaseMerge`; reject transitions to `PhaseReassess` with the reason as `last_error`.

### `/sf agent <subcommand>`

Persistent agent management:

- `/sf agent list` — show all agents with state, last_active, capabilities.
- `/sf agent run <name> "message"` — wake an agent with an ad-hoc message (bypasses inbox routing).
- `/sf agent reset <name>` — clear hot cache and reset Budget; memory blocks and message history preserved.
- `/sf agent delete <name>` — soft-delete (sets `archived_at`); runs and messages preserved via snap_ columns.
- `/sf agent inspect <name>` — show memory blocks, recent messages, current state.
- `/sf agent history <name>` — query archived inbox in `.sf/archive/agents/{id}/`.

### `/sf history [filters]`

Query archived units in `.sf/archive/`. Filter syntax:

```
/sf history --since 2026-04-01 --phase merge --verdict success
/sf history --workflow spike
/sf history --model claude-sonnet-4-6 --limit 50
/sf history --json   # machine-readable output for automation
```

Filters are AND-combined. Without filters, returns the most recent 20 archived units. The query reads from `runs` table joined with archive metadata; full unit artifacts are accessible at `.sf/archive/{date}-{unit-id}/`.

### `/sf clean [--dry-run]`

Garbage-collect: rotate trace JSONL older than 30 days to `.sf/archive/trace/`, evict `pending_retain` rows older than 7 days to `lost-learnings.jsonl`, vacuum SQLite. `--dry-run` shows what would be removed.

---

## 26. Conformance Checklist

Use this checklist as the definition-of-done for each build phase. An implementation is **core-conformant** when all core items pass. **Extension-conformant** when all extension items also pass.

Each item is tagged:

- **[REQUIRED]** — MUST be present for conformance at its tier. Absence = non-conformant.
- **[STRONG]** — SHOULD be present; departure requires a written rationale.
- **[OPTIONAL]** — MAY be present; absence is acceptable.

Default tag is **[REQUIRED]** unless explicitly noted.

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
- [ ] **C-11** Compaction: write session summary to Singularity Memory, clear hot cache, start next turn with fresh recall.
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
- [ ] **C-34** Intent chapters: open/close with intent summary; used for crash recovery context and Singularity Memory recall.
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
- [ ] **C-46** SQLite is orchestration-only — no `memories` table, no vector index. Knowledge MUST live in Singularity Memory.
- [ ] **C-47** Atomic claim acquisition: single conditional UPDATE pattern; rows_affected = 1 gates dispatch.
- [ ] **C-48** `runs` table: CHECK constraint enforces XOR between unit_attempt and agent_run; aggregate token/cost are end-of-run rollup.
- [ ] **C-49** `units.attempt` is current counter; historical attempts in `runs`; both updated in same transaction.
- [ ] **C-50** Tracker `unknown` mid-run does NOT cancel in-flight attempts; only suppresses new dispatch. `blocked` mid-run also non-cancelling.
- [ ] **C-51** Singularity Memory retain failures queue in `pending_retain`; flush to `lost-learnings.jsonl` after 7d.
- [ ] **C-52** Workflow selection priority: tracker label `sf:workflow=` → `default_workflow` config → built-in fallback. Pinned to unit at first dispatch; never re-evaluated.
- [ ] **C-53** PhaseUAT trigger: workflow `require_uat = true`; halts auto-loop with `SignalPause`; resumes via `/sf uat-approve` or `/sf uat-reject`.
- [ ] **C-54** Agent run termination conditions defined (inbox drain, stop tool, hard budget, turn cap, supervisor abort, timeout); hot cache NOT preserved across runs; durable blocks and message history ARE.
- [ ] **C-55** `last_error` injected only on `TurnFirst` of `attempt >= 2`.
- [ ] **C-56** Per-project lock at `<project>/.sf/run.lock`; multiple projects can run auto concurrently.
- [ ] **C-57** Project DB at `<project>/.sf/sf.db`; canonical directory layout (§ 14.5) MUST be honoured for `/sf revert`, `/sf history`, archive sweeps.
- [ ] **C-58** All runtime ULID PKs; soft-delete via `archived_at` for units and agents (no cascade delete of runs).
- [ ] **C-59** `runs` snap_ columns survive entity deletion; FK uses `ON DELETE SET NULL`.
- [ ] **C-60** Per-hook-type timeouts (table in § 10.3); not a single global value.
- [ ] **C-61** PhaseReassess outcomes: Re-plan / Abandon / Escalate; `max_reassess` decrements only on Re-plan; reasoning tier with Think.
- [ ] **C-62** PhaseChange is non-vetoable; veto semantics live on PreDispatch.
- [ ] **C-63** PhaseReview three-pass: establish-context → parallel chunked review → synthesis.
- [ ] **C-64** SSH disconnect: `error_code = "ssh_disconnected"`; remote zombie cleanup via marker pgrep; host quarantine on cleanup failure; orphaned workspace preserved for forensics.
- [ ] **C-65** Agent compaction preserves wake message + recent 3 inbox arrivals + full memory blocks.
- [ ] **C-66** PreToolUse hook decisions outrank auto_approve list (deny wins; allow falls through to auto-approve).
- [ ] **C-67** Slice merge ordering: `code_depends_on` honoured; merges serialised per project.
- [ ] **C-68** Doc-sync sub-step runs at end of last code-mutating phase (Merge if present, else Execute).
- [ ] **C-69** Cost stored as `cost_micro_usd` INTEGER (1e-6 USD); float drift avoided.
- [ ] **C-70** `session_blockers.resolved_at` set per resolution-rules table; `resolved_by` records source.
- [ ] **C-71** Workflow content pinning via `workflow_pins(hash, name, content)`; in-flight units use pinned content even if template file changes.
- [ ] **C-72** `projectHash` derivation: git-remote SHA-256 → fallback path SHA-256; cached in `.sf/runtime/project-hash.json`.
- [ ] **C-73** Dynamic reload of session-immutable fields: warn, keep in-process value, surface in `/sf status` as drift; do NOT crash.
- [ ] **C-74** `last_error` capped at 4 KB head-and-tail; full payload at `.sf/active/{unit-id}/last-error-full.txt`.
- [ ] **C-75** SSH auth via agent / explicit key; `ssh_known_hosts` MUST verify; no auto-trust.
- [ ] **C-76** UAT phase has timeout = 0 (infinite); advanced via `/sf uat-approve` or `/sf uat-reject`.
- [ ] **C-77** HTTP API requires `Authorization: Bearer <token>` from `.sf/runtime/api.token` (mode 0600); `?session=<id>` filter supported.
- [ ] **C-78** `/sf doctor` exit code 0 = all pass, 1 = any FAIL or WARN; `--json` returns structured report.
- [ ] **C-79** Trace JSONL has `_meta` first-line record with `trace_schema_version`; readers branch on version.
- [ ] **C-80** Trace SQL index (`trace_index`) populated by trace writer; `/sf forensics` queries it for fast span lookup.
- [ ] **C-81** Turn outcome marker parsed from last 200 chars: `<turn_status>complete|blocked|giving_up</turn_status>`; blocked → SignalPause, giving_up → PhaseReassess.
- [ ] **C-82** Agent handoff supports `capability:tag1,tag2` form; round-robin by `last_active` among matching agents; `ErrNoCapableAgent` if none.
- [ ] **C-83** Provider API keys MUST use `vault://`; plaintext rejected at startup validation.
- [ ] **C-84** Gate script protocol: env vars (SF_PROJECT_ROOT, SF_UNIT_ID, SF_RUN_ID, SF_PHASE, SF_ATTEMPT, SF_GATE_NAME, SF_GATE_RETRY, SF_WORKSPACE, SF_TRACE_FILE), stdin = UnitResult JSON, exit codes 0/1/2/3, output truncated at 8 KB.
- [ ] **C-85** Gate retry counter is separate from `units.attempt`; resets on phase transition.
- [ ] **C-86** `plan.md` frontmatter (unit_id, created_at, written_by, plan_version) + sections (Goal, Approach, Deliverables, Verification, Notes) validated before transition out of PhasePlan.
- [ ] **C-87** `Memory` interface (Recall, Retain, Feedback, Validate, Health) generated from `singularity-memory/openapi.yaml`; `pending_retain` queue routes failed Retains; `local_anti_patterns` mirror exposed when sm unreachable.
- [ ] **C-88** SF tools registered through Crush's `internal/agent/tools/`; PreToolUse hooks apply uniformly; auto_approve keys documented per tool.
- [ ] **C-89** All operator commands referenced elsewhere in spec are present in § 25: reassess-resolve, force-clear, merge-resolve, uat-approve, uat-reject, agent {list,run,reset,delete,inspect,history}, history, clean.
- [ ] **C-90** `agent_capabilities` index maintained in sync with `agents.capabilities`; capability lookup is index scan, not full table scan.
- [ ] **C-91** Trace JSONL archive move is transactional with `trace_index.file_path` UPDATE; recoverable if interrupted.
- [ ] **C-92** Versioning policy: SemVer; v1.0 freezes §§3, 4, 6, 10, 14, 26.
- [ ] **C-93** [STRONG] Rate-limit data is observability-only; no orchestrator retry/dispatch logic reads it.
- [ ] **C-94** Singularity Memory is the sole knowledge backend; engine assimilated into `singularity_memory_server/` (MIT-attributed, no upstream runtime dep).
- [ ] **C-95** `[memory] mode = "embedded"` is the default for single-user sf; `mode = "remote"` MUST require `url` and `api_key` (vault://).
- [ ] **C-96** Go client `singularity-memory-client-go` is generated from `singularity-memory/openapi.yaml`; sf imports it as a normal Go module dependency.

### 26.2 Knowledge layer (ship after core)

- [ ] **K-01** Memory tiers: hot cache (in-memory, last 10 turns); Singularity Memory store (durable, PostUnit writes).
- [ ] **K-02** Two-bank pattern in Singularity Memory: `project/{hash}` + `global/coding`; merged before dispatch.
- [ ] **K-03** Anti-pattern library: `collection: anti_patterns`; never decay; surfaced in dedicated `<anti_patterns>` block.
- [ ] **K-04** Pattern maturation: 4 states (candidate → established → proven → deprecated); weights as specified.
- [ ] **K-05** Confidence decay: `halfLife = 90 * (0.5 + confidence)` days.
- [ ] **K-06** Singularity Memory is the sole knowledge backend; on sm outage, dispatch proceeds with empty recall (plus local_anti_patterns mirror) and a logged warning.
- [ ] **K-07** `sf init` deep analysis default; `--quick` skips Singularity Memory indexing.

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
