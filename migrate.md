# singularity-crush — Migration Notes

## What this is

**singularity-crush is Crush on autopilot.**

Crush is an interactive coding agent — you drive it turn by turn. singularity-crush adds the autopilot layer on top: when pointed at an existing codebase it maps it, builds the harness, and populates the knowledge store — then drives itself through research → plan → execute → verify → complete without human intervention per unit.

Concretely, `sf init` in an existing project:
1. Indexes the codebase into SQLite (FTS5 + Qwen3-Embedding-4B vectors)
2. Extracts initial patterns and conventions into the memory store
3. Sets up `.sf/config.toml` with project-specific harness config
4. Establishes the session contract and runs doctor checks

Then `sf auto` takes over — the harness drives phase transitions, the supervisor watches for problems, the knowledge layer injects relevant context before each unit, and the model router picks the right model for each phase. The human watches or steers; the agent executes.

This is a fork of [charmbracelet/crush](https://github.com/charmbracelet/crush) as the foundation for porting [singularity-foundry](../singularity-foundry) (SF) from TypeScript to Go. SF is a mature AI coding agent orchestration CLI (v2.75+, ~254k lines of TypeScript). Rather than porting line-by-line, the plan is to build SF's orchestration layer on top of what Crush already provides — Crush is the interactive agent, singularity-crush is the autopilot.

## Why

- Node.js startup latency makes SF feel slow — Go binary is near-instant
- Single binary, no `node_modules`, trivially Nix-packageable
- Crush already implements the agent loop, tool execution, multi-provider LLM, MCP — that's the hard part
- Works natively in Termux, any platform Go runs on

## What Crush already provides (don't rebuild)

- Agent loop via `charm.land/fantasy`
- Multi-provider LLM: Anthropic, OpenAI, Gemini, Groq, Bedrock, Azure, Ollama, etc. via `charm.land/catwalk`
- MCP client support (`modelcontextprotocol/go-sdk`)
- LSP integration for code intelligence
- SQLite state via `ncruces/go-sqlite3`
- Bubbletea TUI, Lipgloss styling, full charmbracelet ecosystem
- Tool execution (bash, file read/write, grep, web search, sourcegraph)

## What SF adds that needs to be built

### Core (build first)

1. **Planning system** — milestones → slices → tasks hierarchy, stored in SQLite
2. **Phase dispatch** — research → plan → execute → complete → reassess loop (`/sf auto`, `/sf dispatch`)
3. **Git/worktree management** — branch-per-slice, clean PR branches, worktree isolation
4. **Session state** — auto-loop persistence, crash recovery, interrupted session resume
5. **Step mode** — `/sf next` for manual stepping through the execution loop

### Important (build second)

6. **Workflow templates** — bugfix, feature, hotfix, spike, refactor, security-audit, dep-upgrade
7. **Parallel orchestration** — parallel slice execution, merge, conflict detection
8. **Knowledge base** — replaced by Hivemind memory layer (see below)
9. ~~**Skill system**~~ — **already done, free from Crush** (see below)
10. **Ship** — PR creation from milestone artifacts

### Persistent agents + inter-agent messaging (build third)

11. **Persistent agent identity** — named agents with stable IDs, system prompts, memory blocks, and message history in SQLite. An agent wakes when its inbox has undelivered messages, runs until idle, then sleeps at zero cost. See `harness.md` § 15.
12. **Memory blocks** — labeled, character-limited blocks (`persona`, `human`, `task`, custom) rendered as XML into the system prompt at dispatch. Two built-in tools: `core_memory_append(label, content)` and `core_memory_replace(label, old, new)`. Block writes commit to SQLite mid-conversation — crash-safe. See `harness.md` § 15.
13. **Inter-agent messaging** — `send_message(to, message)` tool routes a message to a named agent's inbox. Target agent wakes, processes it as a `user` role message, and can reply. `wait_for_reply()` lets the caller block until the reply arrives (with a timeout). Supervisor monitors all running agents independently. See `harness.md` § 16.

### Nice-to-have (build later or drop)

- `/sf visualize` — rebuild lighter with Bubbletea (10-tab TUI is overkill)
- `/sf cmux` — drop entirely (tmux integration, not needed)
- `/sf remote` — Slack/Discord control, handled by Notifier plugin interface
- `/sf migrate` — one-time v1 migration tool, low priority

## Prompt template contract

Every dispatch renders the unit's prompt template with a strict variable checker — unknown variables fail rendering immediately (not silently). Template input variables:

| Variable | Type | Value |
|---|---|---|
| `unit` | object | Full unit record: id, type, phase, title, description, labels, blockers |
| `attempt` | integer or null | `null` on first dispatch; `1+` on retry or continuation |
| `phase` | string | Current phase name (`execute`, `tdd`, etc.) |
| `session_id` | string | Stable session UUID |

The `attempt` variable is the key one: the prompt template can give different instructions to a retrying agent vs. a fresh start. A retry prompt might say "your previous attempt failed with: {{last_error}} — focus on that specifically." The harness injects `last_error` automatically on `attempt >= 1`.

Continuation turns (same thread, subsequent dispatch after a successful turn) receive a short continuation-guidance prompt, not the full task prompt. The full prompt is already in the thread history — resending it inflates context and confuses the model.

## HTTP observability API

The harness exposes a lightweight HTTP server on `localhost` when `server.port` is set in `.sf/config.toml`. It is observability-only — orchestrator correctness never depends on it.

```toml
[server]
port = 7842   # 0 = ephemeral port for tests
```

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
    "input_tokens": 84000, "output_tokens": 12000,
    "cost_usd": 1.24, "seconds_running": 4820
  }
}
```

**`GET /api/v1/units/<unit_id>`** — per-unit debug detail including recent events, workspace path, retry count, last error, and log file path.

**`POST /api/v1/refresh`** — queue an immediate poll + reconciliation cycle (202 Accepted, best-effort coalescing of rapid requests).

Hot-rebind on port change is not required — restart is acceptable for server config changes.

## `/sf revert` — git-aware revert

Four-phase protocol replacing the naive current undo:

**Phase 1 — target selection:**
- If the user provides a target (`/sf revert m1/s2/t3`), go directly to confirmation.
- Otherwise present the top 3 in-progress units + 3 most recently completed units as a numbered menu. User picks one.

**Phase 2 — git reconciliation:**
- Find all commits belonging to the target unit from the activity log and git history.
- Handle ghost commits: if a commit SHA is missing (rebase/squash rewrote history), search by commit message prefix rather than SHA. Log which SHAs were resolved vs. not found.
- Find plan-update commits (commits that modified `.sf/` artifact files for this unit).
- Compile final ordered SHA list (most recent first).

**Phase 3 — confirmation:**
- Display the exact SHA list with descriptions and dates.
- Warn if any SHAs are merge commits (requires `--mainline`).
- Require explicit user confirmation before touching git state.

**Phase 4 — execution:**
- `git revert --no-edit <sha>` in reverse order (newest first).
- On conflict: surface via `SignalPause`, halt, wait for user to resolve.
- After all reverts: restore `.sf/active/{unit-id}/` artifacts from archive, mark unit as `[ ]` in the plan, remove from `completed-units.json`.

```go
type RevertTarget struct {
    UnitID   string
    UnitType string
    SHAs     []string        // resolved from activity log + git log
    Ghosts   []string        // SHAs that couldn't be found — log but continue
    PlanSHAs []string        // plan-update commits to also revert
}
```

## Dispatch scheduling

### Priority ordering

When multiple units are eligible for dispatch, the harness sorts them:

1. **Explicit priority** — tasks with `priority: 1` (urgent) before `priority: 4` (low); unset sorts last
2. **Blocker-free first** — tasks with no unresolved upstream blockers before blocked tasks
3. **Phase order** — earlier phases first (Research before Execute) within the same priority
4. **Created-at** — oldest tasks first as tie-breaker

This order is evaluated fresh on each poll tick — a task that was blocked can become unblocked if its upstream completes between ticks.

### Blocker-aware dispatch

A task is not dispatched if any of its upstream dependencies are **non-terminal**. Terminal means `PhaseComplete`, `PhaseReassess` (resolved), or explicitly cancelled. A dependency stuck in `PhaseVerify` is non-terminal and blocks downstream dispatch. A dependency that failed and was marked abandoned is terminal and does NOT block downstream.

The harness checks blockers from the plan's `blocked_by` list before each dispatch. Blocked tasks stay queued and are re-evaluated on the next tick — no exponential backoff, no retry counter increment. This means blocker resolution is near-instant when the upstream unit completes.

```sql
CREATE TABLE task_blockers (
    task_id     TEXT NOT NULL,
    blocked_by  TEXT NOT NULL,   -- task_id of the upstream dependency
    PRIMARY KEY (task_id, blocked_by)
);
```

### Startup cleanup

On startup, the harness scans `.sf/active/` for unit artifacts whose tasks are now in terminal states (complete, cancelled, superseded). Stale active artifacts are moved to `.sf/archive/` automatically. This prevents workspace accumulation across restarts.

## `/sf status` — project health snapshot

Structured output format (TUI panel + `/sf status` CLI output):

```
Project: singularity-foundry
Phase:   Execute  [m2/s3/t1 — add trace export]
Next:    TDD      [m2/s3/t1]
Blocker: none

Milestones:  2 / 5  (40%)
Slices:      7 / 18 (39%)
Tasks:      14 / 42 (33%)

Session:  4h 12m  |  $0.83  |  claude-sonnet-4-6
```

Blockers surface automatically: a `GateBlocked` event, a `MergeConflict` event, or a supervisor `SignalPause` all write to a `session_blockers` SQLite table. `/sf status` reads this table; the TUI subscribes to the pubsub event directly.

## Skills vs workflow templates — critical distinction

**Skills are inspirational.** A `SKILL.md` file is prompt guidance injected into the agent's context. The agent reads it, uses it as reference, and may follow it loosely. Good for: coding conventions, debugging approaches, domain knowledge, tool usage patterns. The agent can deviate if it judges differently.

**Workflow templates are enforced.** A workflow template (bugfix, feature, hotfix, spike, refactor, security-audit, dep-upgrade) defines the exact phase sequence the harness executes. The harness follows it programmatically — it is not a suggestion to the AI. The AI operates within the template's phase structure; it cannot skip phases or reorder them.

This means workflow templates are **not skills**. They are structured data (TOML/YAML) the harness loads and enforces. The harness reads the template, constructs the phase state machine from it, and drives execution. The agent has no say in whether the template is followed.

```toml
# .sf/workflows/bugfix.toml
name    = "bugfix"
phases  = ["research", "plan", "execute", "tdd", "verify", "review", "merge", "complete"]
max_reassess    = 2
require_tdd     = true
require_review  = true
max_retries     = 3

# .sf/workflows/spike.toml
name    = "spike"
phases  = ["research", "plan", "execute", "complete"]
require_tdd     = false
require_review  = false
max_retries     = 0
```

## Go plugin extension points

When Crush's plugin system stabilises (issue #2038 — Caddy-style compile-time Go modules), singularity-crush should expose these interfaces as plugin boundaries. Each is a clean interface with at least two real implementations that different users genuinely need.

### Plugin interfaces

**`SupervisorCheck`** — already defined in `harness.md`. Custom checks without forking: compliance gates, security scanners, cost limits, organisation-specific policies.

```go
type SupervisorCheck interface {
    Name() string
    Check(ctx context.Context, state SupervisorState) SupervisorSignal
}
```

**`Shipper`** — PR/MR creation. GitHub is the default; GitLab, Bitbucket, Forgejo, Gitea are real needs.

```go
type Shipper interface {
    Ship(ctx context.Context, opts ShipOptions) (ShipResult, error)
}
```

**`VCS`** — version control backend. `git` default, `jj` (Jujutsu) as first alternative (ace already prefers jj).

```go
type VCS interface {
    Commit(ctx context.Context, msg string, files []string) error
    Branch(ctx context.Context, name string) error
    Push(ctx context.Context, remote, branch string) error
    // ...
}
```

**`Store`** — storage backend. SQLite for personal use, PostgreSQL for team/shared sessions — same binary, different plugin.

```go
type Store interface {
    SaveSession(ctx context.Context, s Session) error
    LoadSession(ctx context.Context, id string) (Session, error)
    SaveMemory(ctx context.Context, m Memory) error
    SearchMemory(ctx context.Context, q MemoryQuery) ([]Memory, error)
    // ...
}
```

**`Notifier`** — notification provider. Replaces SF's `/sf remote`. Slack, Discord, webhook — each a plugin.

```go
type Notifier interface {
    Notify(ctx context.Context, event Event) error
}
```

### What stays out of plugins

- Workflow templates — enforced TOML/YAML data, not plugin code
- Skills — SKILL.md files, prompt guidance, not Go code
- Model routing — config + thin Go function + SQLite history (see below)
- Phase transitions — harness-owned, not extensible by design

## Model routing

Not a plugin — a pure function `(phase, complexity, history) → tier` with a feedback loop. No external dependencies, no algorithm variation needed.

### Extended thinking — already in Crush

Crush upstream has full extended thinking support — no cherry-picking needed:
- `Think` / `reasoning_effort` per model in config
- Anthropic extended thinking with budget allocation and beta header
- Gemini thinking levels
- Expandable thinking UI with sidebar
- Streaming thinking deltas

The `reasoning` tier (research, plan phases) should have `Think: true` in config. The harness sets this via the model routing decision — the agent doesn't control whether thinking is enabled.

### Three tiers, multiple models each

Each tier holds multiple candidate models — the router picks within the tier using benchmark scores, not static config:

```toml
[tiers.fast]
models = ["claude-haiku-4-5", "gemini-flash-2.0", "gpt-4o-mini"]

[tiers.standard]
models = ["claude-sonnet-4-6", "gpt-4o", "gemini-2.0-pro"]

[tiers.reasoning]
models = ["claude-opus-4-7", "o3", "gemini-ultra-2.0"]
```

### Phase → tier (static, config-driven)

```toml
[routing]
research = "reasoning"  # Think: true
plan     = "reasoning"  # Think: true
execute  = "standard"
tdd      = "standard"   # writing tests: same tier as coding
verify   = "fast"       # running gates, reading results: fast is fine
review   = "standard"   # structured self-review: needs judgment
merge    = "fast"       # commit message + PR description: fast is fine
complete = "fast"
reassess = "reasoning"  # failure diagnosis: needs depth
memory   = "fast"
```

### Benchmarking — model selection within tier

Within a tier, the router picks the best model based on benchmark scores stored in SQLite. Benchmarks run against real task samples (not synthetic) and record:

- **Quality score** — outcome rating (pass/fail/partial, user `/sf rate` signal)
- **Latency** — time to first token + completion time
- **Cost** — tokens × model price
- **Task fingerprint** — phase + complexity bucket + project type

The benchmark system (inspired by SF's `benchmark-selector.ts` and ace's benchmark daemon) runs periodically and on demand (`/sf benchmark`). Results decay over time — model capabilities change with provider updates.

```go
type BenchmarkResult struct {
    Model       string
    Tier        string
    Fingerprint string    // phase+complexity+project hash
    Quality     float64   // 0.0-1.0
    LatencyP50  time.Duration
    CostPer1k   float64
    SampleCount int
    RecordedAt  time.Time
}
```

Router selection within tier:
```
score = quality * 0.6 + (1 - normalised_latency) * 0.2 + (1 - normalised_cost) * 0.2
```
Weights are configurable. Fallback to tier's first model if no benchmark data exists.

### Complexity upgrade (dynamic)

Classifier at dispatch time — file count, scope breadth, cross-cutting changes → complexity score. Crosses threshold → tier bumps one level. Fingerprint + upgrade decision stored in SQLite for future routing.

### `/sf rate` feedback loop

Two sources depending on mode:

- **Auto-mode** — LLM self-evaluates at unit close: did the output meet the phase objective? Signals `over/ok/under` based on its own assessment of quality vs cost. No human in the loop.
- **Step/interactive mode** — human signals `over/ok/under` after reviewing the unit output.

Both write the same benchmark result format with quality score (`over=0.3`, `ok=0.8`, `under=0.0` effectively blocks model for this fingerprint). Human ratings carry higher weight than LLM self-ratings — configurable multiplier.

### Why not a plugin

The routing decision is `(phase, complexity, benchmark_history) → model`. Pure function, no external dependencies, no algorithm variation needed. Config + SQLite + a thin Go scorer. If a user needs different weights or different tiers, they change TOML — they do not recompile.

## Memory and knowledge — Hindsight

SF's flat `KNOWLEDGE.md` and the planned Hivemind Go implementation are both replaced by **Hindsight** ([vectorize-io/hindsight](https://github.com/vectorize-io/hindsight-cookbook)) — a production memory service already used by hermes-memory.

**Why Hindsight instead of building it:**
- hermes-memory already integrates with it (vendors `hindsight_api` + `hindsight_client`)
- Go client available (`hindsight-client-go`) — no new language needed
- Four parallel retrieval strategies: semantic + keyword (BM25) + graph + temporal — more capable than sqlite-vec + FTS5 alone
- All tools share the same memory banks — patterns from singularity-crush sessions are available in Hermes and ace and vice versa
- Backed by pg0 (PostgreSQL) with VectorChord for vectors

**Integration:** use `hindsight-client-go` in singularity-crush. Pre-dispatch: `recall`. Post-unit: `retain`. Periodically: `reflect` to synthesise insights from accumulated memories.

**SQLite in singularity-crush** then only holds: session state, planning data, phase transitions, benchmark scores. Not knowledge.

Reference: [syntax-syndicate/opencode-swarm-tools](https://github.com/syntax-syndicate/opencode-swarm-tools) Hivemind — design inspiration for maturation and decay patterns, even though Hindsight is the actual backend.

### Memory tiers

Two explicit tiers prevent token bloat during long-running sessions:

**Hot cache** — last 10 turns of the current unit, held in memory (not SQLite). Fast path for immediate reasoning. Never injected into the next unit's context — it's the live conversation window, not persistent memory.

**Hindsight store** — the durable layer. PostUnit writes summaries, learnings, and anti-patterns here. Pre-dispatch reads the top-N most relevant entries. This is what Hindsight manages.

The harness never mixes the two. When compaction fires (budget at 80%), the hot cache is summarised and the summary is written to Hindsight as a `session_summary` entry — the hot cache is then cleared. The next dispatch starts with a fresh context window seeded by Hindsight recall, not a truncated version of the old window.

### Anti-pattern library (correct once, learns forever)

Beyond normal memory entries, Hindsight maintains a separate `anti_patterns` collection. Entries here are:
- Written explicitly when the agent makes a mistake and is corrected (either by a gate failure or by user feedback)
- Tagged `collection: anti_patterns` with `is_negative: true`
- Retrieved at dispatch time alongside normal memories, but presented in a dedicated block: `<anti_patterns>avoid these mistakes...</anti_patterns>`
- Never subject to normal maturation decay — anti-patterns persist at full weight until explicitly removed

The distinction matters: normal memories decay when not accessed (good patterns that stopped being relevant fade). Anti-patterns don't decay — a mistake made once is worth surfacing indefinitely.

```go
type AntiPattern struct {
    ID          string
    Description string    // what went wrong
    Context     string    // when/where this applies
    CorrectPath string    // what to do instead
    SourceUnit  string    // unit ID where the mistake occurred
    CreatedAt   time.Time
}
```

### Storage

**sqlite-vec** with `F32_BLOB(2560)` columns in the same SQLite DB Crush already uses (`ncruces/go-sqlite3`). No extra processes, no separate index files. FTS5 virtual table alongside for BM25 hybrid search — FTS5 is the fallback when the embedding endpoint is unreachable.

**Performance is more than adequate for this use case.** SQLite is the right choice for a single-user CLI tool: sub-millisecond FTS5 queries, ANN vector search handles up to ~1M vectors, a large codebase index is ~100k chunks at most. PostgreSQL + pgvector (what ace uses) is the right choice for multi-user production infrastructure — not for a binary that needs to start in milliseconds. Always enable WAL mode:

```go
db.Exec("PRAGMA journal_mode=WAL")
db.Exec("PRAGMA synchronous=NORMAL") // safe with WAL, significantly faster writes
```

WAL allows concurrent reads during writes — important when parallel workers write memories while the main loop reads them.

Embedding via `https://llm-gateway.centralcloud.com/v1/embeddings`, reranking via `/v1/rerank`. OpenAI-compatible API — use `charmbracelet/openai-go` already in go.mod, no new dependency. API key from Vault:

Both embedding and reranking come in two sizes — use the right one for the context:

| Operation | Model | When |
|---|---|---|
| Query embedding | `qwen3-embedding-0.6B` | Real-time, during retrieval |
| Index embedding | `qwen3-embedding-4B` (2560 dims) | Building codebase index, storing memories |
| Routine rerank | `Qwen3-Reranker-0.6B` | Auto-loop queries |
| Pre-dispatch rerank | `Qwen3-Reranker-4B` | Context injection before each unit — quality matters most here |

```json
{ "embedding_api_key": "vault://secret/singularity-crush#inference_fabric_key" }
```

Graceful degradation to FTS5-only if the endpoint is unreachable (offline, tailnet down).

### Pattern maturation

Every stored knowledge entry has a maturity state:

| State | Condition | Weight in retrieval |
|---|---|---|
| Candidate | < 3 observations | 0.5x |
| Established | ≥ 3 obs, harmful ratio < 30% | 1.0x |
| Proven | decayed helpful score ≥ 5, harmful ratio < 15% | 1.5x |
| Deprecated | harmful ratio > 30% | 0x (excluded) |

Anti-pattern detection: after 3 failed uses, content is prefixed `AVOID:` and flagged `is_negative: true`. Excluded from retrieval by default, surfaced explicitly on request.

### Confidence decay

Exponential half-life decay based on confidence at store time:

```
halfLife   = 90 * (0.5 + confidence)   // days; confidence ∈ [0.0, 1.0]
decayFactor = 0.5 ^ (ageInDays / halfLife)
finalScore  = similarityScore * decayFactor
```

This means:
- High-confidence (1.0): 135-day half-life — slow decay
- Default (0.7): 108-day half-life
- Low-confidence (0.0): 45-day half-life — fast decay

Memory access tiers used for filtering:
- **Hot**: accessed within 7 days
- **Warm**: accessed within 30 days
- **Cold / Stale**: older

Frequency bonus: entries with 10+ accesses gain a 7-day buffer against decay. Validation reset: when a memory directly aids task completion, calling `validate()` resets the 90-day decay timer.

### Schema (extend Crush's `internal/db/`)

```sql
CREATE TABLE memories (
    id           TEXT PRIMARY KEY,
    content      TEXT NOT NULL,
    embedding    F32_BLOB(2560),  -- Qwen3-Embedding-4B
    decay_factor REAL DEFAULT 1.0,
    confidence   REAL DEFAULT 0.7,     -- affects half-life
    maturity     TEXT DEFAULT 'candidate',
    is_negative  INTEGER DEFAULT 0,
    helpful_hits INTEGER DEFAULT 0,
    harmful_hits INTEGER DEFAULT 0,
    access_count INTEGER DEFAULT 0,
    collection   TEXT DEFAULT 'default',
    tags         TEXT,                 -- JSON array
    last_accessed TEXT,
    valid_until  TEXT,
    superseded_by TEXT,
    created_at   TEXT NOT NULL,
    updated_at   TEXT NOT NULL
);

CREATE VIRTUAL TABLE memories_fts USING fts5(content, tags, content=memories);
CREATE INDEX memories_vector ON memories USING libsql_vector_idx(embedding);
```

### Retrieval

Two-stage pipeline — fast broad retrieval then accurate reranking:

**Stage 1 — retrieval (broad, top 20-50 candidates):**
1. Generate query embedding via `llm-gateway.centralcloud.com/v1/embeddings` (Qwen3-Embedding-0.6B — fast, real-time)
2. ANN search via `vector_top_k()` — cosine distance → similarity score
3. FTS5 search in parallel — `bm25()` is BM25, built into SQLite, no extra library
4. **Reciprocal Rank Fusion**: `rrf = 1/(60 + vector_rank) + 1/(60 + fts5_rank)`

**Stage 2 — reranking (precise, top 5-10 returned):**
5. Send (query, document) pairs to reranker at `llm-gateway.centralcloud.com/v1/rerank`
   - **0.6B** (`Qwen/Qwen3-Reranker-0.6B`) — default, fast, used for most queries
   - **4B** (`Qwen/Qwen3-Reranker-4B`) — accurate, used for pre-dispatch context injection where quality matters most
6. Apply decay: `finalScore = rerankerScore * decayFactor`
7. Filter deprecated (`0x`) and below threshold
8. Apply maturity weights
9. Return top N sorted by finalScore

Cross-encoders (rerankers) understand query-document relationships far better than bi-encoder similarity alone. The two-stage approach keeps latency low — reranker only sees 20-50 candidates, not the full store.

Pre-dispatch auto-recall: before each unit dispatch, the harness queries `collection: 'default'` filtered to `hot` tier, injects top 3 as context. This replaces SF's static `KNOWLEDGE.md` injection.

### What this replaces in SF

- `KNOWLEDGE.md` — replaced by the memories table, `collection: 'knowledge'`
- `/sf knowledge add` — `memory.Store(content, confidence)`
- `/sf codebase rag` — `memory.Search(query)` over `collection: 'codebase'`
- `context-injector.ts` — harness pre-dispatch hook reads hot memories and injects

## SSH / Wishlist integration (bonus)

Since this is Go and we're on Headscale:

- Wrap with `tailscale.com/tsnet` to expose singularity-crush as a tailnet node
- Use `github.com/charmbracelet/wish` to serve it as an SSH app
- Use `github.com/charmbracelet/wishlist` for cross-node session directory

This gives browser + Termux + ET access to the agent from anywhere on the tailnet without any extra ingress.

## Secret management — Vault instead of env vars

Crush stores API keys as plaintext in `~/.local/share/crush/crush.json` or resolves `$ENV_VAR`. Both are unacceptable. singularity-crush will use Vault (already running at `vault.hugo.dk` in k3s) as the sole source of secrets.

### Config syntax

Replace `api_key: "$ENV_VAR"` with a `vault://` URI scheme:

```json
{
  "providers": {
    "anthropic": { "api_key": "vault://secret/singularity-crush#anthropic_api_key" },
    "openai":    { "api_key": "vault://secret/singularity-crush#openai_api_key" }
  }
}
```

### Implementation

Extend `internal/config/resolve.go` — add a `VaultResolver` alongside the existing `ShellVariableResolver`:

```go
type VaultResolver struct {
    client *vault.Client   // github.com/hashicorp/vault/api
}

func (r *VaultResolver) Resolve(uri string) (string, error) {
    // parse vault://path#field
    // client.KVv2(mount).Get(ctx, path) → secret.Data["field"]
}
```

Auth chain (first that succeeds):
1. `VAULT_TOKEN` env var (CI/ephemeral)
2. `~/.vault-token` file (local dev, written by `vault login`)
3. AppRole via `VAULT_ROLE_ID` + `VAULT_SECRET_ID` (production/automated)

Secrets are fetched **once at startup** and held in memory for the session lifetime — never written to disk, never logged (Crush's existing redaction in `crush_logs.go` already covers `api_key` fields).

### Stopgap (works today without code changes)

Crush already supports `$(command)` substitution:
```json
{ "api_key": "$(vault kv get -field=anthropic_api_key secret/singularity-crush)" }
```
Use this until the native resolver is built.

### What not to do

- Do not use External Secrets → k8s Secret → env var. That's three hops back to env vars.
- Do not use Vault Agent sidecar for a CLI tool — that's k8s infrastructure, not a local binary.
- Do not store any secret in `crush.json` in plaintext, even temporarily.

## Module rename

Change `github.com/charmbracelet/crush` → `github.com/singularity-ng/singularity-crush` in go.mod and all imports.

## Skill system — already done

Crush implements the [Agent Skills open standard](https://agentskills.io) in `internal/skills/`. Nothing to build.

**Built-in skills Crush ships:**
- `crush-config` — configuration help (crush.json, providers, LSP, MCP, hooks)
- `crush-hooks` — hook authoring (PreToolUse, matchers, allow/deny/halt, input rewriting)
- `jq` — built-in JSON processor via gojq

**Auto-discovered skill paths** (no config needed):
- `.agents/skills/` — **ace-coder's skills already live here, fully compatible**
- `.crush/skills/`
- `.claude/skills/`
- `.cursor/skills/`

Each skill is a directory with a `SKILL.md` file containing YAML frontmatter (`name`, `description`, `license`, `compatibility`) and markdown instructions. SF's TypeScript skill catalog is replaced entirely by dropping `SKILL.md` files into `.agents/skills/`.

**SF skills to port as SKILL.md files** (drop into `.agents/skills/singularity-crush/`):
- `sf-auto` — autonomous mode instructions
- `sf-plan` — planning and phase dispatch
- `sf-git` — worktree and branch conventions
- `sf-knowledge` — when and how to store memories
- `sf-ship` — PR creation conventions

These replace the TypeScript skill catalog, marketplace discovery, and skill manifest system entirely.

## charmbracelet/x packages to use

Experimental packages from [charmbracelet/x](https://github.com/charmbracelet/x) — no backward compat guarantees, but worth pulling for singularity-crush.

**Directly needed:**

| Package | Use |
|---|---|
| `x/exp/teatest` | Testing Bubbletea TUI components without a real terminal |
| `x/vcr` | HTTP recording/playback — test LLM API calls without hitting real providers |
| `x/xpty` | Cross-platform PTY — worktree shell execution, running commands in isolation |
| `x/gitignore` | Gitignore pattern matching — codebase indexer respects `.gitignore` |
| `x/sshkey` | SSH key parsing — Wish/tsnet SSH integration |

**Worth evaluating before committing to raw Bubbletea:**

| Package | Use |
|---|---|
| `x/pony` | Declarative terminal UI markup language — could replace a lot of Bubbletea boilerplate for `/sf visualize` and the status dashboard. Check maturity before deciding. |
| `x/vt` | Virtual terminal emulator — relevant if session replay or terminal recording is wanted |

`teatest` + `vcr` together form the core test harness: reproducible TUI tests and reproducible LLM interaction tests with no real API or terminal needed.

## What to keep from Crush unchanged

- `internal/lsp/` — LSP client, keep as-is
- `internal/db/` — SQLite schema, extend for SF planning tables
- `internal/agent/tools/` — all tools, add SF-specific ones
- `charm.land/fantasy` and `charm.land/catwalk` — do not replace

## Implementation conformance checklist

Derived from Symphony SPEC.md § 18.1. Use as definition-of-done for each build phase.

**Core (must ship):**
- [ ] Workflow path: explicit arg + cwd default (`WORKFLOW.md`)
- [ ] TOML/YAML config loader with typed defaults and `$VAR` resolution
- [ ] Dynamic config watch/reload/re-apply without restart (fsnotify)
- [ ] Polling orchestrator with single-authority in-memory state
- [ ] Issue/task tracker client: candidate fetch + state refresh + terminal fetch
- [ ] Workspace manager: sanitized per-unit paths, creation + reuse
- [ ] Workspace lifecycle hooks: `after_create`, `before_run`, `after_run`, `before_remove`
- [ ] Hook timeout config (default 60s), fatal vs. best-effort semantics per hook type
- [ ] Agent subprocess client: JSON line protocol, session start, turn loop
- [ ] Strict prompt rendering: `unit`, `attempt`, `phase` variables; unknown vars = error
- [ ] Continuation turns: guidance-only prompt on turn > 1, same thread
- [ ] Issue state re-check between turns: break if non-active
- [ ] Exponential retry queue with continuation retry (1s) after normal exit
- [ ] Configurable retry backoff cap (default 5m)
- [ ] Reconciliation: stop runs on terminal/non-active state transitions
- [ ] Workspace cleanup on terminal state (startup sweep + active transition)
- [ ] Blocker-aware dispatch: non-terminal upstream blocks dispatch
- [ ] Priority sort: priority asc → created_at asc → id lexicographic
- [ ] Per-phase concurrency caps (`max_agents_by_phase`)
- [ ] `turn_input_required` → hard failure in auto-mode
- [ ] Unsupported tool call → structured failure result, continue session
- [ ] Structured logs with `unit_id`, `session_id`, `turn_count` context fields
- [ ] Typed error codes (never match on error strings)
- [ ] Startup terminal artifact cleanup

**Extensions (ship after core):**
- [ ] HTTP observability API (`/api/v1/state`, `/api/v1/units/<id>`, `POST /api/v1/refresh`)
- [ ] SSH worker extension (`worker.ssh_hosts`, remote workspace, per-host concurrency)
- [ ] Durable retry queue across restarts (SQLite-backed)
- [ ] `tracker_query` client-side tool (agent reads/updates task state via orchestrator auth)
- [ ] Pluggable tracker adapters beyond Linear (GitHub Issues, Jira, plain SQLite)

## Effort estimate

- Foundation (module rename, SF commands scaffolded): 1-2 days
- Core planning + dispatch + git + auto-loop: 3-5 weeks
- Workflow templates + parallel + ship command: 2-3 weeks
- Hindsight knowledge layer (sqlite-vec + FTS5/BM25 + RRF fusion + maturation): 2-3 weeks
- SF skills as SKILL.md files: 3-5 days
- Persistent agents + memory blocks: 1-2 weeks
- Inter-agent messaging (inbox, wake/sleep, wait_for_reply): 1-2 weeks
- Polish, /sf visualize rebuild: 1-2 weeks

**Total for a working SF-equivalent with persistent agents: ~11-14 weeks** — skill system and agent loop are free from Crush; knowledge layer and agent fleet are net-new capability SF never had.

## Open design decisions

**Hindsight deployment** — must be running and accessible on the tailnet before the memory layer works. Deploy in k3s alongside other services before starting singularity-crush development.

**Two-bank pattern per session** — Hindsight supports arbitrary bank IDs. Two banks, queried separately and merged before each dispatch:

```go
projectRecall := hindsight.Recall("project/"+projectHash, query)
globalRecall  := hindsight.Recall("global/coding", query)
// merge, deduplicate, inject into unit context
```

Project bank holds codebase-specific knowledge. Global bank holds cross-project patterns. No native cross-bank federated search in Hindsight — two calls merged in Go is trivial.

**Compaction = retain + fresh context** — when context hits 80%, dump the session summary into Hindsight (`retain`) and start a fresh context window with a `recall` of the most relevant memories. Better than lossy summarisation — session history becomes searchable memory rather than a truncated digest.

**`sf init` — deep analysis is default, not opt-in.** Shallow init defeats the autopilot. Default `sf init` runs full deep analysis:
1. AST-level codebase scan (languages, structure, entry points, dependencies)
2. Git history analysis (active areas, recent changes, contributors)
3. Retain findings into `project/{hash}` Hindsight bank
4. Establish `.sf/config.toml` with detected stack, workflow templates, model routing hints

`--quick` flag skips the Hindsight indexing for throwaway sessions. Deep is always the default.

**Parallel workers + Hindsight** — concurrent `retain` calls from parallel slice workers use `document_id` (Hindsight's upsert key) derived from content hash. Duplicate memories from parallel workers silently overwrite rather than accumulate.

**Nix flake from day one** — `flake.nix` in the repo root from the first commit. Single Go binary + Nix = installable everywhere on the infrastructure in one line, including k3s nodes via the tailnet.

## Product management via skills (phase 2)

singularity-crush is a coding autopilot first. PM capabilities come later via the skills ecosystem — no code needed, just the right skill packs dropped into `.agents/skills/`.

**Ready-made PM skill packs:**
- [anthropics/knowledge-work-plugins](https://github.com/anthropics/knowledge-work-plugins) — official Anthropic PM skills: `competitive-brief`, `metrics-review`, `product-brainstorming`, `roadmap-update`, `sprint-planning`, `stakeholder-update`, `synthesize-research`, `write-spec`
- [deanpeters/Product-Manager-Skills](https://github.com/deanpeters/Product-Manager-Skills) — 47 PM skills, full workflow from discovery to delivery
- [product-on-purpose/pm-skills](https://github.com/product-on-purpose/pm-skills) — 38 skills covering Triple Diamond (discover → define → develop → deliver)

**Large curated collections:**
- [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills) — 1000+ skills
- [SkillsMP](https://skillsmp.com) — 66,500+ skills
- [LobeHub](https://lobehub.com/skills) — 15,000+ skills

**How it works:** PM skills are inspirational guidance — the agent reads them and uses PM frameworks (roadmaps, sprint planning, specs) on demand. For enforced PM workflows (vision → feature tree → HTDAG) that's ace's domain via MCP. singularity-crush sits between: richer than plain Crush with PM skills loaded, lighter than ace for day-to-day coding work.

**Connection to ace:** When ace is connected via MCP, it replaces the PM skill layer entirely — ace's PM agent owns vision decomposition and task routing, singularity-crush executes. PM skills are for standalone use without ace.
