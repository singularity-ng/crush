# Harness Engineering Practices

Engineering practices for the singularity-crush agent harness. The goal is to define clean boundaries from day one so practices that were bolted onto SF over time (dispatch-guard, exec-sandbox, auto-recovery, context-budget, auto-supervisor, abandon-detect — all separate files, all added later) are instead structural from the start.

## What Crush already provides

Before adding anything, understand what exists:

- `internal/hooks/` — PreToolUse hook system with allow/deny/halt decisions, input rewriting, multi-hook aggregation. **Use this as-is.** It is well-designed.
- `internal/permission/` — Permission service with pubsub, persistent session grants, hook pre-approval via context key. **Extend, don't replace.**
- `internal/pubsub/` — Generic event broker used across hooks, permissions, notifications. **Use for all SF events.**
- `internal/session/` — Session lifecycle.
- `internal/db/` — SQLite via sqlc. **Extend schema here for SF planning tables.**
- `internal/agent/` — Agent loop, tool execution, fantasy integration.
- `internal/event/` — Event types.

## Harness boundary

The harness is the layer between the agent loop (fantasy) and SF's orchestration logic (milestones, phases, git, worktrees). It owns:

1. Context budget
2. Phase transitions
3. Unit lifecycle hooks
4. Session contract (crash recovery, resume)
5. Observability
6. Supervision (stuck loop, timeout, abandon)

Nothing in the planning or git layers should reach past the harness boundary into fantasy directly.

## 1. Context budget

Context budget is a first-class harness concern, not an application concern.

Define a `Budget` type owned by the harness:

```go
type Budget struct {
    MaxTokens     int
    UsedTokens    int
    CompactAt     float64 // fraction, e.g. 0.80
    HardLimitAt   float64 // fraction, e.g. 0.95
}

func (b *Budget) ShouldCompact() bool { return float64(b.UsedTokens)/float64(b.MaxTokens) >= b.CompactAt }
func (b *Budget) AtHardLimit() bool   { return float64(b.UsedTokens)/float64(b.MaxTokens) >= b.HardLimitAt }
```

Rules:
- The harness updates `UsedTokens` after every model response — the agent loop never manages this itself.
- When `ShouldCompact()` is true the harness triggers compaction before the next dispatch, not mid-turn.
- When `AtHardLimit()` the harness halts the current unit, snapshots state, and surfaces an error — it does not let the agent proceed and hit a provider error.
- Budget state is persisted to SQLite after every turn so crash recovery can restore it.

## 2. Phase transitions

Phases (research → plan → execute → complete → reassess) are harness-owned state machine transitions, not ad-hoc function calls.

```go
type Phase int

const (
    PhaseResearch Phase = iota // map the problem, gather context
    PhasePlan                  // decompose into slices and tasks, get sign-off
    PhaseExecute               // write the code
    PhaseTDD                   // write tests for what was just built; red → green
    PhaseVerify                // run full test suite + lint + type check; gates pass
    PhaseReview                // structured self-review: correctness, style, security
    PhaseMerge                 // commit, push, open PR — git service handles this via PostUnit hook
    PhaseComplete              // unit done; benchmark result recorded
    PhaseReassess              // something failed; re-enter planning with failure context
    PhaseUAT                   // human acceptance; only used when uat_dispatch = true
)

type PhaseTransition struct {
    From      Phase
    To        Phase
    UnitID    string
    Timestamp time.Time
    Reason    string
}
```

Standard flow: `Research → Plan → Execute → TDD → Verify → Review → Merge → Complete`

On gate failure in Verify: `Verify → Execute` (retry, up to `max_retries`). On max retries exceeded: `Verify → Reassess`. On review finding a real problem: `Review → Execute`. On merge conflict: `Merge → Reassess`.

**Run attempt lifecycle** — within each phase, individual dispatch attempts move through finer-grained states:

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
    AttemptStalled        // stall_timeout exceeded since last agent event
    AttemptCanceled       // issue became non-active mid-run (reconciliation)
)
```

`AttemptCanceled` is distinct from `AttemptFailed` — it means the work was valid but the task was externally invalidated (deleted, moved to a terminal state, superseded). The harness does not retry a canceled attempt; it releases the slot and moves on.

**Continuation turns** — after a turn completes, the harness re-checks the unit's eligibility before dispatching the next turn. If still eligible, the continuation turn receives only a short guidance prompt injected into the existing thread context — not the full original task prompt that is already in thread history. This avoids prompt inflation on long multi-turn tasks.

```go
type TurnKind int

const (
    TurnFirst        TurnKind = iota // full rendered task prompt
    TurnContinuation                 // continuation guidance only, same thread
)
```

Workflow templates control which phases are active. A `spike` template might be `Research → Plan → Execute → Complete` with no TDD, Verify, or Merge. A `bugfix` template runs the full sequence. The harness enforces whatever sequence the template defines — it does not infer phases.

```toml
# .sf/workflows/feature.toml
name    = "feature"
phases  = ["research", "plan", "execute", "tdd", "verify", "review", "merge", "complete"]
require_tdd     = true   # PhaseTDD is enforced; skipping it is a gate violation
require_review  = true
max_retries     = 3      # applies per gate in PhaseVerify
```

**Prompt template variable enforcement:**
Prompt templates use `{{variable}}` placeholders. The renderer operates in strict mode: if any `{{variable}}` in the template has no corresponding key in the vars map, `loadPrompt` panics at startup rather than silently rendering an empty string. This catches template/code drift immediately rather than at dispatch time when the missing variable would silently produce a malformed prompt.

The canonical template variables for execute-task are:

| Variable | Type | Notes |
|---|---|---|
| `unit_id` | string | Stable identifier for this unit |
| `unit_type` | string | "milestone" \| "slice" \| "task" |
| `phase` | string | Current phase name |
| `attempt` | int \| null | null on first dispatch; integer (≥ 1) on retry |
| `issue` | object | Full issue/task struct as a flat map |

When adding a new `{{variable}}` to any prompt template: (1) pass it in every `loadPrompt` call site, (2) add a placeholder value in every test that renders that template, (3) recompile. Skipping step (1) or (2) will cause a startup panic in the next run.

Rules:
- All phase transitions go through a single `Harness.Transition(ctx, from, to, reason)` method.
- Transition emits a pubsub event (extend `internal/pubsub`). The TUI subscribes; it never polls state.
- Transition is persisted to SQLite before the new phase begins. A crash mid-phase means on resume the harness sees the last committed transition and re-enters that phase cleanly.
- Invalid transitions (e.g. `Execute → Research` without a `Reassess`) are rejected at the harness boundary with a typed error — not silently allowed and corrected downstream.
- The harness sets `Think: true` on the model config for `Research` and `Plan` phases; off for all others. The agent does not control this.

## 3. Unit lifecycle hooks

Extend Crush's existing `internal/hooks/` with SF-specific hook events alongside `PreToolUse`:

```go
const (
    EventPreToolUse   = "PreToolUse"   // already in Crush
    EventPreDispatch  = "PreDispatch"  // before a unit is dispatched
    EventPostUnit     = "PostUnit"     // after a unit completes
    EventPhaseChange  = "PhaseChange"  // on phase transition
    EventAutoLoop     = "AutoLoop"     // each auto-mode iteration
)
```

The existing hook aggregation logic (`aggregate()` in `hooks.go`) handles allow/deny/halt — reuse it for all new events.

**Unsupported tool calls — continue, don't stall:**
If the agent calls a tool that is not registered or not supported in the current dispatch context, the harness returns a structured tool failure result to the agent and continues the session. It never stalls or panics on an unknown tool name. The agent sees the failure in its context and can adapt. This applies to both built-in tools and dynamic tools registered via hooks.

**Client-side tool response contract:**
Every tool call — successful or not — receives a response in this shape:

```go
type ToolResponse struct {
    Success      bool          `json:"success"`
    Output       string        `json:"output"`
    ContentItems []ContentItem `json:"contentItems"`
}

type ContentItem struct {
    Type string `json:"type"`  // always "inputText" for text results
    Text string `json:"text"`
}
```

For successful calls: `success = true`, `output` = result summary, `contentItems` contains the full result text. For unsupported or failed calls: `success = false`, `output` = human-readable error, `contentItems` lists which tools *are* supported in the current context. This shape must be consistent — the agent relies on the `success` field to distinguish real failures from tool-not-found errors. Never return a bare string or a different shape for error cases.

PostUnit hooks receive a `UnitResult` payload: phase, duration, token cost, model used, verdict. This is what enables `/sf rate` (model tier rating) and post-unit git operations without the harness knowing about git.

**Doc sync hook (built-in, runs after PhaseMerge):**
After a slice or milestone merges, the harness dispatches a lightweight doc-sync unit. The agent checks whether the completed work changed the tech stack, introduced a new pattern, or requires a guideline update, and proposes edits to project-level docs (`ARCHITECTURE.md`, `CONVENTIONS.md`, `STACK.md`). The agent writes proposed changes as a diff to stdout; the harness surfaces them to the TUI for user approval before committing. This keeps project context current without manual maintenance.

Rules:
- Doc sync runs as a `fast`-tier dispatch (short context, light reasoning needed).
- It only runs after `PhaseMerge`, not after every phase.
- If the agent finds nothing to update it emits an empty diff and the hook is a no-op.
- Doc sync can be disabled per-project: `[harness] doc_sync = false`.

## 4. Session contract

A session has a defined contract the harness enforces:

```go
type SessionState int

const (
    SessionIdle SessionState = iota
    SessionRunning
    SessionPaused
    SessionInterrupted  // abnormal stop, recoverable
    SessionComplete
    SessionFailed       // unrecoverable
)
```

Rules:
- Only the harness writes session state to SQLite — never the planning or execution layers.
- On startup, the harness checks for `SessionInterrupted` state. If found it offers resume before accepting new work.
- `SessionPaused` is a clean pause (`/sf pause`) — state is fully committed, resume is safe.
- `SessionInterrupted` means the process died mid-turn. The harness reconstructs from the last committed phase transition and unit state. It does not replay tool calls.
- A lock file (`~/.sf/run.lock` containing PID) prevents two SF processes writing to the same session. The harness acquires this on start and releases on clean exit. On startup, a stale lock (PID not running) is cleaned up automatically.

## 5. Observability

Every harness boundary crossing is traced. Use structured logging via Crush's existing `internal/log` package and extend with span-style events:

```go
type Span struct {
    TraceID   string
    SpanID    string
    Operation string       // "tool_call", "phase_transition", "model_request"
    StartedAt time.Time
    Duration  time.Duration
    Attrs     map[string]any
    Error     error
}
```

**Structured log format:**
All harness log lines use stable `key=value` phrasing. Required context fields:

| Scope | Required fields |
|-------|----------------|
| Any unit-related log | `unit_id=`, `unit_type=` |
| Agent session lifecycle | `session_id=`, `turn_count=` |
| Phase transitions | `from=`, `to=`, `reason=` |
| Gate execution | `gate=`, `attempt=`, `passed=` |

Include action outcome in the message: `completed`, `failed`, `retrying`, `canceled`. Never log large raw payloads — truncate at 512 bytes and note `[truncated]`. If a log sink fails, continue running and emit a warning through any remaining sink.

Rules:
- Every tool call, phase transition, model request, and hook execution emits a span.
- Spans are written to a JSONL file (`~/.sf/trace.jsonl`) that rolls daily. This is what `/sf forensics` and `/sf logs debug` read.
- Span emission is synchronous but non-blocking — use a buffered channel with a background writer goroutine. Never drop a span on the floor; if the buffer is full, block briefly rather than discard.
- Cost tracking (tokens × model price) is computed at the harness layer from model response metadata, not estimated by the agent. `/sf session-report` reads directly from the trace.

**Log rotation:**
Disk log is bounded: 10MB max file size, 5 rotating files. Lines are single-line (no multi-line JSON blobs). When file logging is active, the default stderr/console handler is removed — logs go to file only. Default path: `~/.sf/log/sf.log`. Configurable:

```toml
[harness.log]
path       = "~/.sf/log/sf.log"
max_size   = 10_485_760   # 10MB in bytes
max_files  = 5
stderr     = false        # true = write to both file and stderr (dev only)
```

Never write raw multi-line hook output to the structured log — truncate at 2 KB and append `(truncated)` if longer.

**Token accounting precision:**
Provider responses can arrive as absolute thread totals or as per-turn deltas. The harness always prefers absolute totals when available (`thread/tokenUsage/updated`-style events) and tracks the last-reported total to compute deltas, preventing double-counting if both payload types appear in the same session. Never treat a generic `usage` map as a cumulative total unless the event type explicitly defines it that way. Aggregate totals (input, output, cache-read, cache-write, cost-usd) accumulate in the orchestrator state and are included in every runtime snapshot.

**Rate-limit tracking:**
The harness tracks the latest rate-limit payload from any provider event and surfaces it in the TUI and HTTP API. No retry logic is driven by this — the circuit breaker and backoff handle that separately. Rate-limit data is observability-only.

**`turn_input_required` in auto-mode:**
When the agent raises `turn_input_required` during auto-mode, there are two valid responses:

- **Soft response (preferred):** inject a fixed message — `"This is a non-interactive session. Operator input is unavailable."` — as a `user` role turn and let the session continue. The agent sees the message and can adapt (e.g. make a reasonable default choice and continue). This keeps the session alive and avoids unnecessary retry noise for agents that ask a clarifying question but don't strictly require an answer.
- **Hard failure:** end the attempt immediately, record `ErrTurnInputRequired`, schedule a normal failure retry. Appropriate when the project requires zero ambiguous agent choices in auto-mode.

The default behaviour is configurable per-project:

```toml
[harness]
turn_input_required = "soft"   # or "hard"
```

In interactive/step mode, the harness always surfaces the request to the user via the TUI and waits up to `unit_timeout` before failing with `ErrTurnInputRequired`. The harness MUST NOT leave a run stalled indefinitely.

**Intent chapters:**
Spans are grouped into named chapters — not just by phase, but by intent. A chapter opens when the agent starts pursuing a distinct goal (e.g. "investigate auth bug", "write migration") and closes when that intent resolves (success, failure, or pivot). Chapter boundaries are inferred from phase transitions and tool call patterns; the agent can also explicitly open a chapter via `chapter_open(name)`.

Chapters serve two purposes:
1. **Context recovery** — on resume after a crash, the harness reconstructs "what the agent was doing and why" from the chapter log rather than replaying raw tool calls. The chapter summary is injected at the top of the restored context.
2. **Hindsight recall** — completed chapters are stored as discrete memory units in Hindsight. Recall queries match against chapter intent, not just content, giving more relevant results for "how did we handle X before?"

```go
type Chapter struct {
    ID        string
    UnitID    string
    Name      string        // inferred or agent-declared
    Intent    string        // one-sentence summary written at close
    OpenedAt  time.Time
    ClosedAt  *time.Time
    Outcome   string        // "success" | "failure" | "pivot"
    SpanIDs   []string      // spans that belong to this chapter
}
```

## 6. Supervision

The harness runs a supervisor goroutine alongside the agent loop. It does not touch agent state directly — it communicates via the pubsub broker.

Supervisor checks (run every N seconds during auto-mode):

```go
type SupervisorCheck interface {
    Name() string
    Check(ctx context.Context, state SupervisorState) SupervisorSignal
}

type SupervisorSignal int
const (
    SignalOK SupervisorSignal = iota
    SignalWarn      // log, surface in TUI
    SignalPause     // pause auto-loop, wait for user
    SignalAbort     // stop unit, mark interrupted
)
```

Built-in checks:
- **StuckLoop** — same phase for > N turns with no tool calls completing successfully
- **BudgetWarning** — context approaching compaction threshold
- **TimeoutCheck** — unit running longer than configured max
- **AbandonDetect** — agent producing output with no tool calls (talking without acting)
- **GitDivergence** — working branch has diverged from base in unexpected ways
- **ReconciliationCancel** — the unit's underlying issue/task transitioned to a non-active state (deleted, cancelled, superseded) while the agent was running. The supervisor emits `SignalAbort` with reason `canceled_by_reconciliation`; the harness transitions to `AttemptCanceled` and releases the slot without retry.
- **BlockerCheck** — before each dispatch, the harness checks whether the unit has unresolved blockers (upstream tasks not yet `PhaseComplete`). If blockers exist, dispatch is skipped and the unit stays queued. Checked again on the next poll tick, not retried with backoff.
- **ModelUnavailable** — provider returns a "model not supported / not found" error class (distinct from a transient 429 or 5xx). On detection the supervisor emits `SignalAbort` immediately rather than retrying to the timeout; the harness surfaces a typed `ErrModelUnavailable` so the router can fall back to the next candidate in the tier rather than spinning.
- **CircuitBreaker** — when the same model fails with non-transient errors 3 consecutive times within a session, the supervisor trips a circuit breaker for that model for the remainder of the session. Subsequent dispatches in that tier skip the tripped model. The circuit state is written to SQLite so it survives a restart. Resets after 24h or on explicit `/sf reset-circuits`.

The supervisor signals map to pubsub events the TUI and auto-loop both subscribe to. The auto-loop acts on `SignalPause` and `SignalAbort`; the TUI shows warnings on `SignalWarn`. The supervisor never calls `os.Exit` or panics — it signals and the harness decides.

## 7. Tool sandboxing

Extend Crush's existing permission service (`internal/permission/`) rather than building a new sandbox.

SF-specific permission rules:
- `git:write` — any git operation that mutates state (commit, branch, push). Requires explicit grant in auto-mode.
- `worktree:create` and `worktree:delete` — worktree lifecycle operations.
- `fs:write-outside-project` — writes outside the project root. Always prompt, never auto-approve.
- `shell:exec` — arbitrary shell execution. Allowlist specific commands rather than blanket approval.

In auto-mode the harness calls `permission.AutoApproveSession()` only for the allowlisted operations defined in the project's `.sf/config.toml`. Sensitive operations always prompt regardless of auto-mode.

## 8. Configuration contract

Harness configuration lives in `.sf/config.toml` (project-level) and `~/.sf/config.toml` (global). Project overrides global.

```toml
[harness]
context_compact_at = 0.80
context_hard_limit = 0.95
unit_timeout = "10m"
supervisor_interval = "10s"

[harness.auto_approve]
tools = ["bash:read", "fs:read", "git:status", "git:diff"]

[harness.hooks]
pre_dispatch = ["./hooks/pre-dispatch.sh"]
post_unit = ["./hooks/post-unit.sh"]
```

Rules:
- The harness validates config on startup and fails fast with a descriptive error — it never silently ignores unknown keys or bad values.
- `/sf doctor` runs `HarnessConfig.Validate()` as one of its checks.
- **Dynamic reload** — the harness polls `.sf/config.toml` every second using a `{mtime, size, content_hash}` stamp rather than fsnotify (simpler, more portable across Linux/macOS/WSL). When the stamp changes, it re-parses and re-validates. Valid changes apply immediately to future dispatch, concurrency limits, and hook lists. Invalid reloads log an error and keep the last known good config — never crash. In-flight agent runs are not interrupted by a config reload.
- The following fields are always session-immutable even with dynamic reload enabled: `worktree_mode`, `context_compact_at`, `context_hard_limit`. Changing these requires restart.

```toml
[harness]
context_compact_at  = 0.80
context_hard_limit  = 0.95
unit_timeout        = "10m"
supervisor_interval = "10s"
doc_sync            = true    # post-merge doc synchronisation hook

[harness.concurrency]
max_agents                    = 10
max_agents_by_phase.execute   = 4   # cap concurrent execute-phase units
max_agents_by_phase.tdd       = 4
max_agents_by_phase.verify    = 10  # verify is cheap; allow more

[harness.auto_approve]
tools = ["bash:read", "fs:read", "git:status", "git:diff"]

[harness.hooks]
pre_dispatch = ["./hooks/pre-dispatch.sh"]
post_unit    = ["./hooks/post-unit.sh"]
```

`max_agents_by_phase` limits how many units can be in a given phase simultaneously across all running agents. This prevents resource contention — e.g. capping concurrent `execute` phases stops 10 agents from all hammering the same codebase at once while `verify` phases (which only read) can run freely.

## 9. Knowledge layer via hermes-memory

The knowledge layer is **hermes-memory**: a Hermes plugin installed at `$HERMES_HOME/plugins/hermes_memory/`. It is not an interface the harness calls directly — Hermes orchestrates the full lifecycle. The harness boundary is narrow: emit the right events with the right payload, let hermes-memory do the rest.

**How it works:**

- Hermes calls `prefetch()` before each dispatch and injects retrieved memories into the agent's system prompt via `system_prompt_block()`. The harness does not manage this — it is automatic once hermes-memory is registered as the active provider.
- After each unit, the harness PostUnit hook (§ 10) can explicitly store learnings by calling the `hermes_memory_store` tool or by including a `learnings` field in the PostUnit payload. hermes-memory's `on_memory_write()` handler persists them.
- After a successful unit, the harness emits a positive feedback signal via `hermes_memory_feedback`. This resets the decay timer on memories that were recalled during that unit.

**Backend:**
Postgres (durable storage), local file (dev and tests). Dense embeddings via `llm-gateway.centralcloud.com/v1/embeddings` (Qwen3 4B, 2560d vectors). Lexical retrieval via BM25. Fused ranking via reciprocal-rank fusion. Optional reranking via `llm-embedding.centralcloud.com/v1`.

**Tools the agent can call directly during a unit:**

| Tool | Purpose |
|------|---------|
| `hermes_memory_search` | Semantic + lexical search across stored memories |
| `hermes_memory_context` | Retrieve context-relevant memories for the current task |
| `hermes_memory_store` | Explicitly store a learning or insight |
| `hermes_memory_feedback` | Signal helpfulness (positive or negative) for a memory |

The agent calling these tools is the primary write path. The harness only calls them from PostUnit hooks — it does not maintain a memory client itself.

**Anti-pattern detection:**
hermes-memory tracks negative feedback counts per entry. After repeated `hermes_memory_feedback` calls with negative signal against the same memory, it marks the entry as a negative pattern. The harness emits the feedback signal after `Verdict: Failure`; hermes-memory owns all decay and flagging logic.

**Hindsight loop:**
After each unit (success or failure), the harness synthesizes a structured hindsight summary and stores it via `hermes_memory_store`. The summary includes: what was attempted, what succeeded, what failed, and what the agent should do differently next time. This is explicitly distinct from the task output — it is a self-reflection record. On failure, the hindsight entry is tagged `outcome:failure` and the anti-pattern decay starts immediately. Future units in the same session retrieve these entries via `hermes_memory_context` at dispatch time so the agent doesn't repeat the same mistakes within a session. Across sessions, the decayed entries surface only if the same context pattern is detected again.

## 10. Post-unit hook pipeline

After each unit completes, the harness serializes a `UnitResult` payload and dispatches it through all registered `PostUnit` hooks in order. Hooks are synchronous from the harness's perspective — they run sequentially, not concurrently, and the next dispatch does not begin until all PostUnit hooks have returned.

```go
type UnitResult struct {
    UnitID       string
    UnitType     string        // "milestone", "slice", "task"
    Phase        Phase
    Verdict      string        // "success", "failure", "abandoned"
    Duration     time.Duration
    InputTokens  int
    OutputTokens int
    CacheHits    int
    CostUSD      float64
    Model        string
    Error        error         // non-nil on failure
    Learnings    []string      // explicit learnings extracted by the agent
}
```

Hook execution rules:
- Each hook runs in a child process (same pattern as `PreDispatch` hooks in config). The `UnitResult` is serialized to JSON and passed via stdin.
- A hook that exits non-zero signals `SignalAbort` — the harness stops the session and marks it `SessionFailed`. A hook that times out (default 30s) is killed and logged but does not block the next dispatch.
- The git service subscribes to PostUnit via a hook and handles commits, branch creation, and push. The harness knows nothing about git.
- hermes-memory feedback is emitted from a built-in PostUnit hook, not a subprocess — it calls `hermes_memory_feedback` and, if `Learnings` is non-empty, calls `hermes_memory_store` for each entry.
- PostUnit hook results are written to the trace as child spans of the unit span.

## 11. Worktree isolation

SF operates in one of two worktree modes, configured per-project in `.sf/config.toml`:

```toml
[harness]
worktree_mode = "branch-per-slice"   # or "milestone-per-worktree"
```

**branch-per-slice** (default):
- Each slice gets its own git branch (`sf/m{n}-s{n}-{slug}`) created from the current base.
- The harness emits `worktree:create` permission events before branch creation; a git service hook handles the actual `git worktree add`.
- After a slice's PostUnit hooks run, the git service merges the branch back to the integration branch. The harness waits for the merge hook to return before marking the slice complete.
- Merge conflicts trigger `SignalPause` — the supervisor surfaces the conflict in the TUI and halts auto-mode until the user resolves it.

**milestone-per-worktree**:
- A single worktree is created for the entire milestone at milestone start.
- All slices in the milestone share that worktree. The git service commits incrementally within it.
- The worktree is merged (or PR'd) at milestone PostUnit time.
- Suitable for milestones where slices are tightly coupled and sequential; branch-per-slice is preferred for parallel-safe work.

Worktree lifecycle events (extend `internal/hooks/`):

```go
const (
    EventWorktreeCreate = "WorktreeCreate"
    EventWorktreeDelete = "WorktreeDelete"
    EventMergeReady     = "MergeReady"
    EventMergeConflict  = "MergeConflict"
)
```

The harness emits these events; the git service subscribes. The harness never calls `git` directly.

## 12. Model routing and budget signals

Model routing is not a harness concern — the harness does not select models. It does, however, emit signals that the router reads:

- **BudgetWarning** (pubsub) — emitted when `ShouldCompact()` is true. The router may downgrade model tier to extend the budget window.
- **AtHardLimit** (pubsub) — emitted before the harness halts the unit. The router records this as a signal that the current model tier consumed the context.
- Each span in the trace records `model` in its attributes. `/sf session-report` reads the trace to report per-model token costs.

The router subscribes to these events independently. The harness does not know what decision the router makes — it only acts on the budget state it observes.

## 13. Verification gates

Between PostUnit and the next dispatch, the harness runs a verification gate. The gate is a list of checks defined in `.sf/config.toml`:

```toml
[harness.gates]
post_slice = ["./gates/run-tests.sh", "./gates/lint.sh"]
post_milestone = ["./gates/integration-tests.sh"]
```

Gate execution rules:
- Gates run as subprocesses. The gate script receives the `UnitResult` JSON on stdin (same payload as PostUnit hooks).
- A gate exits 0 = pass. Non-zero = fail. Fail increments the retry counter for that unit.
- Default max retries: 3. Configurable per gate type (`max_retries_slice`, `max_retries_milestone`).
- On retry, the harness re-dispatches the same unit with the gate failure appended to the context. The agent sees what failed and why.
- After max retries, the harness transitions to `SessionFailed` and surfaces a `GateBlocked` event on pubsub. The TUI shows which gate failed and what output it produced.
- Gate results are stored in SQLite and written as span events on the unit span so they appear in `/sf forensics`.

```go
type GateResult struct {
    GateName  string
    UnitID    string
    Passed    bool
    Attempt   int
    MaxRetries int
    Output    string    // combined stdout+stderr, truncated at 8KB
    Duration  time.Duration
}
```

**PhaseReview — chunked review:**
Large diffs must not be reviewed in a single pass. The harness splits the changed file list into chunks of ≤ 300 lines each before dispatching the review agent. Each chunk is reviewed independently; findings are accumulated. The final review pass synthesises all findings across chunks. This keeps each review dispatch within a predictable context budget and prevents the agent from glossing over the end of a large diff.

```go
const ReviewChunkLines = 300

func chunkDiff(files []ChangedFile) [][]ChangedFile {
    // bin-pack files into chunks, respecting ReviewChunkLines
    // files larger than ReviewChunkLines get their own chunk
}
```

**Unit archive after completion:**
When a slice or milestone reaches `PhaseComplete`, the harness moves its artifact directory from `.sf/active/` to `.sf/archive/{date}-{unit-id}/`. Active work stays in `.sf/active/` (small, fast to scan). Archived units are queryable via `/sf history` and readable by the hindsight recall pipeline. The archive move is atomic (rename, not copy+delete).

**`specs.check` gate (built-in, run at PhaseVerify):**
Every exported Go function, type, and constant in the harness must have a godoc comment. The `specs.check` gate runs an AST-based pass over the harness package and fails if any public identifier is missing documentation. This is a compile-time quality gate, not a style suggestion.

```go
// Gate: exit 1 if any exported decl lacks a doc comment
// Implemented as a go/ast walk — no external linter dependency.
// Only applies to the harness package itself, not to extension or hook code.
```

This keeps the public API self-documenting without relying on a linter that the agent might not have available in the workspace.

## 14. SF environment variables and lock files

### Environment variables

| Variable | Purpose |
|----------|---------|
| `SF_PROJECT_ROOT` | Absolute path to the project root. Set by the harness on startup; never read from the environment — the harness always sets it. |
| `SF_HOME` | SF data directory. Defaults to `~/.sf`. Overridable for multi-user systems. |
| `SF_TRACE_ENABLED` | Set to `1` to enable structured trace collection to `~/.sf/trace.jsonl`. Off by default. |
| `SF_MILESTONE_LOCK` | Set to the active milestone ID when a milestone is in progress. Hook scripts can read this to scope their work. |
| `SF_UNIT_ID` | Set to the active unit ID during dispatch and hook execution. |
| `SF_SESSION_ID` | Set to the harness session UUID. Stable across resume. |

### Lock files

**`~/.sf/run.lock`** — global process lock. Contains the PID of the running SF process. The harness acquires this on startup (fails if another PID is alive and running). On clean exit, the file is removed. On startup, a stale lock (PID not in `/proc`) is cleaned up automatically and logged.

**`.sf/auto.lock`** — signals that auto-mode is active for this project. Contains the session ID. Created when auto-mode starts, removed when it stops (cleanly or via interrupt). Hook scripts and external tools can check for this file to gate behavior.

**`.sf/runtime/paused-session.json`** — written by the harness when `SessionPaused` is committed. Contains: session ID, phase, unit ID, budget state, timestamp. On resume, the harness reads this file, restores state, and deletes the file. If the file exists but the session is not `SessionPaused` (e.g., process was killed), the harness treats it as `SessionInterrupted` and offers recovery.

**`.sf/runtime/gate-state.json`** — written by the harness after each gate run. Contains the last gate result per unit, retry counts, and blocked state. Persisted so that a crash mid-gate retry does not reset the retry counter.

## 15. Persistent agents

An agent is a named, persistent identity with its own memory blocks, system prompt, and message history. Unlike a unit (which is ephemeral work within a session), an agent persists indefinitely and can be woken at any time by an incoming message or an explicit dispatch.

### Schema

```sql
CREATE TABLE agents (
    id          TEXT PRIMARY KEY,         -- stable UUID
    name        TEXT NOT NULL UNIQUE,
    system      TEXT NOT NULL,            -- system prompt template
    model       TEXT NOT NULL,
    created_at  INTEGER NOT NULL,
    last_active INTEGER
);

CREATE TABLE agent_memory_blocks (
    agent_id    TEXT NOT NULL REFERENCES agents(id),
    label       TEXT NOT NULL,            -- e.g. "persona", "human", "task"
    value       TEXT NOT NULL DEFAULT '',
    char_limit  INTEGER NOT NULL DEFAULT 2000,
    read_only   INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (agent_id, label)
);

CREATE TABLE agent_messages (
    id          TEXT PRIMARY KEY,
    agent_id    TEXT NOT NULL REFERENCES agents(id),
    seq         INTEGER NOT NULL,         -- monotonically increasing per agent
    role        TEXT NOT NULL,            -- "user" | "assistant" | "tool_call" | "tool_return" | "system"
    content     TEXT NOT NULL,
    tool_name   TEXT,                     -- set when role = tool_call or tool_return
    created_at  INTEGER NOT NULL
);
```

### Memory block injection

At dispatch time the harness renders the agent's memory blocks into the system prompt:

```
<memory>
<block label="persona">{{value}}</block>
<block label="human">{{value}}</block>
</memory>
```

The agent can edit blocks mid-conversation using two built-in tools:

| Tool | Signature | Effect |
|------|-----------|--------|
| `core_memory_append` | `(label string, content string)` | Appends content to the named block (respects char_limit) |
| `core_memory_replace` | `(label string, old string, new string)` | Replaces a substring within the named block |

Both tools write directly to `agent_memory_blocks` before the next turn is dispatched, so a crash mid-session preserves the updated block state.

### Agent lifecycle

```go
type AgentState int

const (
    AgentIdle    AgentState = iota  // no pending messages, not running
    AgentRunning                    // dispatched, consuming tokens
    AgentWaiting                    // sent a message to another agent, awaiting reply
    AgentStopped                    // explicitly stopped; will not wake automatically
)
```

The harness owns all state transitions. The agent loop never writes `AgentState` directly.

## 16. Inter-agent messaging

Agents communicate by calling a `send_message` tool. The harness routes the message to the target agent's inbox, wakes it if it is `AgentIdle`, and records both the outbound and inbound messages in `agent_messages`.

### Wire format

```go
// Tool the agent calls:
// send_message(to: string, message: string) -> void
//
// to: agent name or agent ID
// message: plain text; the receiving agent sees it as a "user" role message

type AgentMessage struct {
    ID         string
    FromAgent  string
    ToAgent    string
    Content    string
    SentAt     time.Time
}
```

### Inbox and wake

Each agent has a persistent inbox table:

```sql
CREATE TABLE agent_inbox (
    id          TEXT PRIMARY KEY,
    agent_id    TEXT NOT NULL REFERENCES agents(id),
    from_agent  TEXT NOT NULL,
    content     TEXT NOT NULL,
    delivered   INTEGER NOT NULL DEFAULT 0,
    created_at  INTEGER NOT NULL
);
```

Wake rules:
- When `send_message` is called, the harness inserts into `agent_inbox` and emits an `AgentWake` pubsub event for the target agent.
- The target agent's run loop checks its inbox at the start of each dispatch. Undelivered messages are prepended to the context as `user` role messages in arrival order, then marked `delivered = 1`.
- An `AgentIdle` agent that receives an `AgentWake` event starts a new dispatch cycle immediately. An `AgentRunning` agent queues the message for its next cycle.
- An agent that sends a message and calls `wait_for_reply()` transitions to `AgentWaiting`. The harness suspends its dispatch loop until the target agent sends a reply (via another `send_message`) or a configurable timeout elapses.

### Pubsub events

```go
const (
    EventAgentWake    = "AgentWake"     // target agent should start/resume
    EventAgentMessage = "AgentMessage"  // message routed (for TUI and tracing)
    EventAgentIdle    = "AgentIdle"     // agent completed its turn, no pending messages
)
```

### Harness guarantees for agent fleets

Every agent run is wrapped by the same harness that wraps single-agent units:
- Budget tracking and compaction fire per-agent, not globally.
- The supervisor goroutine monitors all running agents; `StuckLoop` and `AbandonDetect` checks apply to each independently.
- Crash recovery restores each agent to its last committed message sequence independently — a crash in agent B does not affect agent A's recovery.
- Cost is recorded per agent in the trace. `/sf session-report` breaks down spend by agent ID.

### Agent handoff

An agent that determines a task is outside its competence can call `handoff(to, context)` to transfer the active task to a named specialist agent. Handoff is different from `send_message`:

- The calling agent's current unit is suspended (not completed).
- The target agent receives the full task context (system prompt, memory blocks, last N messages) pre-loaded into its inbox.
- The calling agent transitions to `AgentWaiting` until the specialist replies with a result or escalates back.
- If the target agent is not found or is `AgentStopped`, `handoff` returns an error and the calling agent continues.

Handoff is the runtime complement to static phase routing — the agent self-nominates when the task complexity exceeds its configured role. A coordinator agent that receives a handoff reply can merge the result and continue its own unit.

```go
// Tool the agent calls:
// handoff(to: string, context: string) -> HandoffTicket
//
// Returns a ticket ID. The agent calls wait_for_reply(ticket_id) to block.
```

### Inbox event log (append-only)

`agent_inbox` is append-only — rows are never deleted or updated, only inserted and marked `delivered`. This gives the harness a complete audit trail of all inter-agent communication. Conflict resolution for shared state happens at read time (last-writer-wins on `agent_memory_blocks` by `updated_at`), not via locking. The event log is the source of truth for what happened; the current block state is a materialised view of it.

### What not to build for inter-agent messaging

- **Shared memory** — agents do not share memory blocks. If two agents need a common fact, one agent sends it as a message. Shared mutable state creates the same race conditions here as anywhere else.
- **Broadcast** — there is no `send_message_all`. A coordinator agent sends individual messages. This keeps routing explicit and traceable.
- **Synchronous RPC** — `send_message` is fire-and-forget from the caller's perspective. `wait_for_reply()` is a separate explicit call, and it has a timeout. Never block the harness loop indefinitely waiting for another agent.

## 17. Failure taxonomy

Every harness failure has a class. The class determines recovery behavior — not the error message.

| Class | Examples | Recovery |
|---|---|---|
| `config` | Missing WORKFLOW.md, invalid TOML, unknown tracker kind, missing API key | Block new dispatches. Keep service alive. Continue reconciliation. Emit operator-visible error. |
| `workspace` | Directory creation failure, hook timeout, invalid path | Fail the current attempt. Orchestrator retries with backoff. |
| `agent_session` | Startup handshake failed, turn timeout, turn cancelled, subprocess exit, stalled session, `turn_input_required` (auto-mode) | Fail the current attempt. Orchestrator retries with backoff. |
| `tracker` | API transport error, non-200, GraphQL errors, malformed payload | **Candidate fetch failure**: skip this tick, try next tick. **Reconciliation failure**: keep workers running, retry next tick. **Startup cleanup failure**: log and continue. |
| `observability` | Snapshot timeout, dashboard render error, log sink failure | Log and ignore. Never crash the orchestrator over an observability failure. |

**Scheduler state is intentionally in-memory.** Restart recovery does not restore retry timers, live sessions, or in-flight agent state. After restart: startup terminal cleanup → fresh poll → re-dispatch eligible work. This is not a limitation — it is a design choice. Durable retry state is on the TODO list for a future conformance extension.

```go
// Typed error codes — never match on error strings
const (
    ErrMissingWorkflowFile       = "missing_workflow_file"
    ErrWorkflowParseError        = "workflow_parse_error"
    ErrUnsupportedTrackerKind    = "unsupported_tracker_kind"
    ErrMissingTrackerKey         = "missing_tracker_api_key"
    ErrWorkspaceCreation         = "workspace_creation_failed"
    ErrHookTimeout               = "hook_timeout"
    ErrAgentStartup              = "agent_session_startup"
    ErrTurnTimeout               = "turn_timeout"
    ErrTurnFailed                = "turn_failed"
    ErrTurnInputRequired         = "turn_input_required"
    ErrStalled                   = "stalled"
    ErrCanceledByReconciliation  = "canceled_by_reconciliation"
    ErrModelUnavailable          = "model_unavailable"
    ErrCircuitOpen               = "circuit_open"
)
```

## 18. Worker attempt inner loop

The exact sequence inside a single worker run — how the harness wires workspace, hooks, agent session, and multi-turn continuation:

```
run_worker_attempt(unit, attempt):
  workspace = create_or_reuse_workspace(unit.id)
  if workspace failed → fail_attempt(ErrWorkspaceCreation)

  run_hook("before_run", workspace) → fatal if fails

  session = agent.start_session(cwd=workspace)
  if session failed:
    run_hook_best_effort("after_run", workspace)
    fail_attempt(ErrAgentStartup)

  turn = 1
  loop:
    prompt = build_prompt(unit, attempt, turn)
    if prompt failed:
      agent.stop_session(session)
      run_hook_best_effort("after_run", workspace)
      fail_attempt(ErrPromptRender)

    result = agent.run_turn(session, prompt)
    if result failed:
      agent.stop_session(session)
      run_hook_best_effort("after_run", workspace)
      fail_attempt(result.error)

    // re-check unit state between turns
    current_state = tracker.fetch_unit_state(unit.id)
    if current_state is non-active → break   // AttemptCanceled
    if turn >= max_turns → break

    turn++
    // continuation turns get guidance-only prompt, not the full original

  agent.stop_session(session)
  run_hook_best_effort("after_run", workspace)
  exit_normal()
```

**After a normal exit**, the orchestrator schedules a short continuation retry (1 second) to re-poll and decide whether the unit is still eligible. This is not a failure retry — it is deliberate re-evaluation. If the unit is still active, a new session starts; if terminal, the claim is released.

**After an abnormal exit**, exponential backoff: `delay = min(10s × 2^(attempt-1), max_retry_backoff)`. Default cap: 5 minutes.

## 19. Distributed execution (SSH worker extension)

The orchestrator always runs centrally. Workers can execute on remote hosts over SSH — relevant for executing agents on tailnet nodes with more RAM/GPU.

```toml
[worker]
ssh_hosts                      = ["mikki-bunker", "forge-gpu-1"]
max_concurrent_agents_per_host = 3
```

Rules:
- `workspace.root` is resolved on the **remote host**, not the orchestrator.
- The agent subprocess is launched over SSH stdio; the orchestrator owns the session lifecycle even though execution is remote.
- Continuation turns within one worker lifetime stay on the same host and workspace.
- If a host is at capacity, dispatch waits rather than silently falling back to local or another host.
- Once a run has produced side effects, moving to another host on retry is treated as a new attempt (not invisible failover).
- Host health: a dead or overloaded host reduces available capacity; it does not cause duplicate execution.
- The run record includes `worker_host` so operators can see where each run executed and where to find its logs.

This is the model for running singularity-crush against the ace-coder or inference-fabric repos on bunker while keeping the orchestrator on the local machine.

## 20. Trust boundary and harness hardening

Every deployment MUST document its trust posture explicitly. There is no universal safe default.

**Singularity-crush defaults (trusted single-user developer machine):**
- Auto-approve tool execution and file changes within the workspace.
- `turn_input_required = "soft"` (inject non-interactive message, let agent adapt).
- Workspace isolation enforced (path containment, sanitized names).
- Secrets from Vault only — never plaintext in config.

**Workspace path containment — symlink-aware canonicalization:**
A naive `filepath.EvalSymlinks` or `path.Clean` check can be defeated by a symlink that resolves outside the workspace root. The harness uses segment-by-segment canonicalization instead:

```go
// resolveCanonical walks each path segment via lstat,
// following symlinks one hop at a time and re-resolving from the new target.
// Only fails on permission errors — ENOENT for not-yet-created paths is OK
// (returns the unconsumed tail appended to the resolved prefix).
func resolveCanonical(root string, segments []string) (string, error) {
    // for each segment, lstat the candidate path:
    //   - symlink → read link target, expand relative to current resolved prefix,
    //                prepend remaining segments, recurse
    //   - regular  → append to resolved prefix, continue
    //   - ENOENT   → append segment + rest as-is (path not yet created)
    //   - other err → return error
}
```

After canonicalization, assert `canonical_workspace` has `canonical_root + "/"` as a prefix. This catches symlinks that escape the root even if `path.Clean` would not. Apply this check in both directions: local filesystem (via `lstat`) and, for remote workers, via a shell script that resolves each segment before `mkdir`.

The workspace directory name is also sanitized: `[^a-zA-Z0-9._-]` → `_`. This prevents path injection via issue identifiers that contain slashes, `..`, or null bytes.

**Hardening measures for less-trusted environments:**
- Filter which issues/tasks are eligible for dispatch — untrusted or out-of-scope tasks must not automatically reach the agent.
- Restrict the `linear_graphql` (or equivalent) client-side tool to read-only or project-scoped mutations only.
- Run the agent subprocess under a dedicated OS user with no write access outside the workspace root.
- Add container or VM isolation around each workspace (Docker, nsjail, etc.).
- Restrict network access from the workspace — the agent should not be able to call arbitrary external APIs.
- Narrow available tools to the minimum needed for the workflow.

Treat harness hardening as part of the core safety model, not an afterthought. Tracker data, repository contents, and tool arguments are not unconditionally trustworthy even when they originate inside a normal workflow.

## What not to build in the harness

- **Git operations** — the harness emits `PostUnit` hooks; a git service subscribes and acts. Git logic is not in the harness.
- **Planning logic** — milestone/slice/task management is above the harness. The harness knows about phases and units, not their content.
- **TUI rendering** — the harness emits pubsub events. The TUI subscribes. The harness never formats output.
- **Model selection** — the harness tracks cost and surfaces budget signals. Model routing is a separate concern that reads those signals.
- **Provider errors** — the harness catches them at the boundary and translates to typed errors. Provider-specific retry logic lives in catwalk, not here.
