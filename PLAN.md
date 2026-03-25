# contextcoder — Implementation Plan

## Context

Building a Rust-based coding CLI with superior context management. The product is designed around one rule:

> Before every LLM call, construct the smallest correct context for that request locally.

Key differentiators:
- Request-time context planning, not additive prompt growth
- Tree-sitter as the local structural index for files, symbols, scopes, imports, and ranges
- LSP as the local semantic source of truth for diagnostics, definitions, references, hover, and rename impact
- Fast-access Markdown files for durable project knowledge, but only as hints alongside code-aware context
- Long-term SQLite-backed learning of project patterns and implementation decisions across languages
- Plan-first workflow enforced at the session layer
- Sandboxed tools (command allowlist, project root boundary)
- Multi-provider AI via configurable TOML config

**Decisions:**
- MVP scope: TUI + required Tree-sitter + required LSP + basic tools + OpenAI-compatible API
- Tree-sitter and LSP are required for supported languages; the app is designed around them, not degraded gracefully without them
- Smart context selection/compression is the next milestone after MVP, not part of the first usable release
- Reuse existing language servers and an existing Rust LSP client crate; do not build a custom JSON-RPC/LSP transport layer
- Missing language support is installed only with explicit user approval at runtime; the app asks, installs, verifies, then continues
- First AI provider: OpenAI-compatible (covers OpenAI + Ollama + LM Studio + llama.cpp)
- Bash sandbox: command allowlist only (macOS-compatible, no landlock needed)
- Embeddings are optional future work; v1 relies on local structure + semantic queries, not full context injection
- Long-term goal: a SQLite knowledge layer that learns your implementation patterns and decisions by language so new projects can start closer to your preferred style

## Roadmap Shape

### Milestone 1 — Usable Core (MVP)
- TUI
- OpenAI-compatible API only
- Required Tree-sitter and LSP support with user-approved installs
- Basic read/edit/list/bash tools
- Plan/todo workflow
- Minimal context assembly only: current request, recent turns, plan state, and explicit tool results

### Milestone 2 — Context Manipulation
- Request-time context selection
- Local symbol/reference/diagnostic indexes
- Compression layer for tool outputs and selected artifacts
- History summarization and context preview

### Milestone 3 — Expansion
- Additional AI providers
- Better persistence and retrieval
- Cross-project learned patterns and decision memory
- Reliability, recovery, and polish

---

## Workspace Structure

```
contextcoder/              ← repo root
├── Cargo.toml             ← workspace manifest
├── contextcoder/          ← binary crate (TUI entry point, session orchestration)
├── cc-config/             ← TOML config + API key management
├── cc-context/            ← Markdown store + SQLite (plans, todos, sessions, decisions)
├── cc-tools/              ← Tool trait + ReadFile, BashSandbox, FetchUrl, TreeSitter, Lsp
└── cc-ai/                 ← AiProvider trait + OpenAI-compat + Anthropic + Gemini + Local
```

---

## Key Dependencies (workspace-level)

| Purpose | Crate |
|---|---|
| TUI | `ratatui`, `crossterm`, `tui-textarea` |
| Async | `tokio` (full), `async-trait`, `futures` |
| HTTP / streaming | `reqwest` (rustls), `eventsource-stream` |
| SQLite | `rusqlite` (bundled), `rusqlite_migration` |
| Tree-sitter | `tree-sitter` with runtime-managed grammars for supported languages |
| LSP | `async-lsp-client`, `lsp-types` |
| Config | `serde`, `toml`, `secrecy` |
| CLI args | `clap` (derive) |
| File watching | `notify` (v6) |
| Token counting | `tiktoken-rs` (accurate budgeting for OpenAI-compatible models); fallback `bytes / 4` estimate for local models until model-specific tokenizers exist |
| Errors | `thiserror`, `anyhow` |
| IDs / time | `uuid` (v4), `chrono` |
| Gitignore-aware walk | `ignore` |

---

## Context Manipulation (Milestone 2, post-MVP)

This is the first major milestone after MVP. The MVP is intentionally simpler: it relies on explicit tools plus a small rolling conversation window. After the usable core is stable, the next step is to make context assembly proactive and local.

The goal is **correct context, not maximum context**. The system should not keep appending files, logs, and history until the window is full. For every user turn, a local planner rebuilds the prompt context from scratch using current intent plus local code intelligence.

### Per-request pipeline

Every LLM call follows this pipeline:

1. Parse the user turn into a `RequestIntent` (`explain`, `edit`, `debug`, `refactor`, `test`, `plan`, etc.)
2. Select candidate files and symbols from the Tree-sitter index
3. Expand and verify those candidates with LSP data (definitions, references, diagnostics, hover, workspace symbols)
4. Build a compact context IR from the selected artifacts
5. Apply budget caps by dropping lower-priority artifacts, not by blindly truncating everything
6. Send only the final request-scoped context to the model

This planner runs before the first model token is requested and again before any follow-up model/tool loop that needs fresh context.

### Missing support flow

For supported languages, missing Tree-sitter grammar support or an unavailable LSP is a blocking prerequisite, but the app should resolve that interactively:

1. Detect the missing dependency during project open or before the first request that needs it
2. Present the user with the exact component to install and the installer that will be used
3. Ask for approval
4. If approved, run the installer, verify the binary/grammar loads correctly, and continue
5. If denied or install fails, stop the request and surface the exact remediation steps

Target interaction:

> TypeScript support needs `typescript-language-server` and the TypeScript Tree-sitter grammar. Install now?

This flow should feel like Claude Code: explicit, interruptible, and resumable after approval.

### Tree-sitter output format (compact, not JSON)

JSON is verbose — field names repeat for every node. All tree-sitter output uses a **compact line-oriented format** designed for token efficiency:

**File structure summary** (from `ReadFile` on large code files):
```
src/session.rs [rust] 312 lines
  fn new(config, context) :12-28
  fn run(&mut self) :30-89
  fn handle_tool_call(&self, call) :91-134
  struct SessionState :136-142
  impl SessionState :144-201
    fn transition(&self, event) :146-178
```
Format: `  <kind> <name>(<params abbreviated>) :<start>-<end>`
Params are abbreviated (names only, no types) to save tokens.

**TreeSitterQuery result** (arbitrary query):
```
match@12: fn handle_event
match@47: fn handle_tool_call
match@91: fn handle_error
```
Format: `match@<line>: <captured text>`

**ReadSymbol result** — returns just the raw source of the named node, no wrapper:
```rust
fn handle_tool_call(&self, call: ToolCall) -> Result<ToolResult> {
    // ... extracted source verbatim
}
```

These formats are defined as Rust formatting functions in `cc-tools/src/tree_sitter/format.rs` and reused by `ReadFile`, `ReadSymbol`, and `TreeSitterQuery`.

---

### Context budget priority (what gets dropped first when a request budget fills)
1. **Keep always:** system prompt, current plan + todo status, current user message
2. **Keep always for supported languages:** selected symbol snippets, relevant imports/module headers, active diagnostics, direct definitions/references
3. **Keep if space:** minimal Markdown context (`project.md`, `patterns.md`, `libs.md`) as hints
4. **Summarize first:** old tool results and stale conversation history
5. **Drop first:** unrelated history, low-confidence related files, decorative logs

### Tool Output Compression Layer (`cc-ai/src/compressor.rs`)

A single `ToolOutputCompressor` sits after local selection and before the LLM message list. Tools return typed, rich output. The compressor decides how selected artifacts are serialized for the model. It is **not** the component that decides which artifacts belong in context.

```
Tool.call() → raw ToolOutput → ToolOutputCompressor → compressed ToolOutput → messages → LLM
```

The compressor applies schema-aware strategies:

| Detected content type | Strategy |
|---|---|
| **Code artifacts** | Compact symbol/range format via `format.rs` plus verbatim snippets only for selected ranges |
| **LSP diagnostics** | Keep path, severity, code, message, and range in a compact line format |
| **LSP navigation data** | Keep symbol name, file path, range, and relation (`def`, `ref`, `impl`) |
| **Generic JSON** | Minify only; do not heuristically delete fields from structured outputs |
| **Log / compiler output** | Keep failing spans, first lines, and tail; insert omission marker |
| **Acknowledgement** (`write_file`, `edit_file`) | Pass through unchanged (already minimal) |
| **Error message** | Pass through unchanged (errors must be fully visible to the AI) |

Structured outputs should be identified by tool type and Rust enums, not only by string heuristics.

Each compressed output appends a token count annotation when truncation occurred:
```
[compressed: 1840 → 210 tokens]
```

The compressor is also used when summarizing old tool results in conversation history — same strategies, just applied to stored messages instead of fresh output.

**Interface:**
```rust
// cc-ai/src/compressor.rs
pub struct ToolOutputCompressor {
    token_counter: Arc<dyn TokenCounter>,
    ts_formatter: Arc<TreeSitterFormatter>,
}
impl ToolOutputCompressor {
    pub fn compress(&self, tool_name: &str, output: ToolOutput, budget: usize) -> ToolOutput;
}
```

The `budget` is passed per-call so `RequestContextPlanner` can apply tighter limits when the window is nearly full.

### FastContext section caps (configurable in `[context]`)
```toml
[context]
max_project_md_tokens = 800
max_patterns_md_tokens = 400
max_libs_md_tokens = 400
max_doc_tokens_per_file = 600   # for user-added library docs
```
If a section exceeds its cap, it is truncated with a note: `[truncated — edit .contextcoder/project.md to reduce size]`.

### Conversation history summarization
Implemented after request-time context selection:
- After every turn, estimate total token count of conversation history
- If `history_tokens > max_context_tokens * 0.6`: summarize the oldest 30% of tool result messages into a single compressed block using a cheap/fast model
- Tool results are the first to be summarized (most repetitive); user/assistant messages are last
- Summaries never replace the request-scoped code context chosen for the current turn

---

## Critical Interfaces (stabilize early)

### `cc-ai/src/provider.rs` — AiProvider trait
```rust
#[async_trait]
pub trait AiProvider: Send + Sync {
    fn name(&self) -> &str;
    async fn stream(
        &self,
        messages: &[Message],
        tools: &[ToolDefinition],
        opts: &CompletionOptions,
    ) -> Result<Pin<Box<dyn Stream<Item = StreamEvent> + Send>>, AiError>;
}

pub enum StreamEvent {
    TextDelta(String),
    ToolCallStart { id: String, name: String },
    ToolCallInputDelta { id: String, partial_json: String },
    ToolCallEnd { id: String },
    Usage { input_tokens: u32, output_tokens: u32 },
    Done,
    Error(AiError),
}
```

### `cc-tools/src/tool.rs` — Tool trait
```rust
#[async_trait]
pub trait Tool: Send + Sync {
    fn name(&self) -> &'static str;
    fn description(&self) -> &'static str;
    fn input_schema(&self) -> serde_json::Value;  // JSON Schema for AI
    async fn call(&self, input: serde_json::Value) -> Result<ToolOutput, ToolError>;
}
// ToolRegistry: HashMap<&str, Box<dyn Tool>> with:
//   dispatch(name, input) -> ToolResult
//   list_definitions() -> Vec<ToolDefinition>  // used by RequestContextPlanner
// Tool failures return ToolOutput::Error(msg), NOT panics — AI receives error and decides next step
```

### `cc-ai/src/request_context.rs` — RequestContextPlanner (Milestone 2)
```rust
pub struct RequestContextPlanner {
    symbol_index: Arc<dyn SymbolIndex>,
    lsp_registry: Arc<LspRegistry>,
    compressor: Arc<ToolOutputCompressor>,
    token_counter: Arc<dyn TokenCounter>,
}

impl RequestContextPlanner {
    pub async fn build(
        &self,
        request: &UserRequest,
        session: &SessionState,
        budget: usize,
    ) -> Result<RequestContext, ContextError>;
}

pub struct RequestContext {
    pub plan_state: String,
    pub selected_symbols: Vec<SelectedSymbol>,
    pub selected_ranges: Vec<SelectedRange>,
    pub diagnostics: Vec<DiagnosticSummary>,
    pub references: Vec<ReferenceSummary>,
    pub fast_context_hints: Vec<ContextHint>,
}
```

### `cc-context/src/manager.rs` — ContextManager facade
```rust
pub struct ContextManager { md_store, db_store, config }
impl ContextManager {
    pub async fn get_fast_context(&self) -> FastContext;   // reads MD files
    pub async fn save_plan(&self, content: &str) -> Plan;
    pub async fn approve_plan(&self, id: i64);
    pub async fn list_todos(&self, plan_id: i64) -> Vec<Todo>;
    pub async fn update_todo_status(&self, id: i64, status: TodoStatus);
}
pub struct FastContext {
    pub project_md: String,
    pub patterns_md: String,
    pub libs_md: String,
    pub extra_docs: Vec<(String, String)>,
}
```

---

## Database Schema (SQLite at `.contextcoder/context.db`)

```sql
sessions    (id, project_path, started_at, ended_at, summary, message_count)
plans       (id, session_id, content, approved, created_at)
todos       (id, plan_id, description, status, order_index, created_at, completed_at)
decisions   (id, title, body, tags_json, created_at, session_id)
symbols     (id, file_path, language, kind, name, parent_name, start_line, end_line, signature, created_at, updated_at)
symbol_edges(id, from_symbol_id, to_symbol_id, edge_kind, created_at)
diagnostics (id, file_path, language, code, severity, message, start_line, start_col, end_line, end_col, updated_at)
```
MVP requires only `sessions`, `plans`, `todos`, and `decisions`.
`symbols`, `symbol_edges`, and `diagnostics` are added in Milestone 2 for local context manipulation.
Long-term, the database should also support cross-project learning tables for reusable patterns, language-specific implementation preferences, and linked decisions.

**SQLite threading rule:** `rusqlite` is synchronous. Every DB call must be dispatched via
`tokio::task::spawn_blocking` — never hold the connection across an `.await` point or the
tokio executor will stall and the TUI will freeze.

---

## Config File (`~/.config/contextcoder/config.toml`)

```toml
[ai]
default_provider = "openai"  # openai | anthropic | gemini | local
default_model = "gpt-4o"
temperature = 0.2
max_tokens = 8192

[providers.openai]
api_key = "sk-..."           # or set OPENAI_API_KEY env var
base_url = "https://api.openai.com/v1"

[providers.anthropic]
api_key = "sk-ant-..."
model = "claude-opus-4-6"

# "local" is not a separate provider implementation — it's an OpenAI-compat
# endpoint with a custom base_url. Define as many as needed:
[providers.ollama]
base_url = "http://localhost:11434/v1"
model = "llama3"

[providers.lmstudio]
base_url = "http://localhost:1234/v1"
model = "local-model"

[context]
max_context_tokens = 128000
max_request_context_tokens = 12000
max_history_tokens = 24000

[languages]
auto_install_support = true
require_treesitter = true
require_lsp = true
prompt_before_install = true

[tools]
allowed_bash_commands = ["cargo", "git", "npm", "pnpm", "yarn", "python", "pytest", "make"]
fetch_allowlist = ["docs.rs", "crates.io", "github.com", "pypi.org", "npmjs.com", "*.github.io"]
bash_timeout_secs = 30

[prompts]
base_system_prompt = ""     # prepended to every system message; empty = built-in default
plan_prompt = ""            # injected when starting planning phase
```

---

## TUI Layout

```
┌─ File Tree ──────┬─── Chat ─────────────────────┬─ Plan / Todos ──┐
│ src/             │  [assistant] Here's the plan  │ ## Plan         │
│ ├── main.rs      │  ...                          │ - [ ] Step 1    │
│ ├── lib.rs       │  [user] approve               │ - [x] Step 2    │
│ └── tools/       │  [assistant] Starting step 1  │                 │
│                  │  ...                          │ Context: 3 chunks│
├──────────────────┴───────────────────────────────┴─────────────────┤
│ > type your message here                    [1200 tok | Planning]  │
└──────────────────────────────────────────────────────────────────────┘
```

Columns: file tree (20%), chat (55%), plan+todos (25%). Bottom bar: input + token count + session state.

---

## Session State Machine

```
Idle → Planning → PlanPending ──/approve──→ Executing → Done
```

- **Planning**: AI calls `create_plan(content)` tool (structured, not text-parsed) to submit plan
- **PlanPending**: plan shown to user awaiting `/approve`; only `read_file`, `read_symbol`, `read_range`, `list_files`, `tree_sitter_query`, and `lsp_query` allowed
- **Executing**: full tool set active, todos tracked in real time
- AI calls `mark_todo_done(order_index)` — uses stable order index, not auto-generated DB ID
- Todo order indices are passed back to AI in context after plan approval so it knows which to reference

---

## Implementation Phases

### Phase 0 — Workspace skeleton
- Root `Cargo.toml` workspace manifest
- Stub all five member crates with empty `lib.rs` / `main.rs`
- All dependency declarations
- `cargo check --workspace` passes

### Phase 1 — Config + bootstrap (`cc-config`)
- `Config` struct, TOML deserialization, `Default` impls for all fields
- API key resolution: config file → env var → `keyring` (OS keychain), using `secrecy::SecretString`
- `clap` CLI: `contextcoder` (launch), `contextcoder init`, `contextcoder config set <key> <value>`
- Bootstrap config for required language support (`auto_install_support`, per-language overrides)

### Phase 2 — Language support registry (`cc-tools`)
- Detect project languages on open and on `init`
- Verify required Tree-sitter grammar and LSP binary for each supported language
- Ask the user before every missing-support install; on approval, install and verify; on denial, stop the request cleanly
- `LanguageSupportRegistry`: language → grammar provider, LSP command, install/check hooks, install strategy, verification hooks, status
- `InstallCoordinator`: owns consent prompts, installer execution, retry state, and post-install verification

### Phase 3 — SQLite + Markdown context (`cc-context`)
- `rusqlite_migration` migrations for MVP tables (`sessions`, `plans`, `todos`, `decisions`)
- `MdStore`: read/write `.contextcoder/*.md`, create default templates on first run
- `DbStore`: CRUD for sessions, plans, todos, decisions
- `ContextManager` facade
- `notify` watcher for live-reload of MD files into `FastContext`

### Phase 4 — Tree-sitter indexing (`cc-tools`)
- Runtime grammar loading for supported languages
- `LanguageRegistry`: file extension → grammar/runtime loader
- Tree-sitter grammar install adapters for supported languages, using curated sources and user-approved install steps
- `format.rs`: compact line-oriented output format; no JSON output for code structure
- `TreeSitterExtractor`: extract top-level and nested symbols, imports, module headers, scopes on demand
- `ReadFile` integration: summary for discovery, exact range reads for explicit tool requests
- `ReadSymbol(file, name)` and `ReadRange(file, start, end)` tools
- `WriteFile` / `EditFile` syntax validation with reparsing

### Phase 5 — LSP integration (`cc-tools`)
- Reuse an existing LSP client crate (`async-lsp-client`) instead of building a custom protocol transport
- `LspRegistry`: one managed server per language/workspace
- LSP install adapters for supported languages (`npm`, `cargo`, `pip`, etc.) with exact command previews shown before execution
- Document lifecycle sync (`didOpen`, `didChange`, `didClose`) for edited buffers
- `LspQuery` tool: hover, diagnostics, go-to-definition, references, rename preview, workspace symbols
- Lazy initialization per language, but execution is blocked for supported languages until required LSP support is available and verified

### Phase 6 — Core tools (`cc-tools`)
- `Tool` trait + `ToolRegistry` (name-based dispatch + `list_definitions()`)
- `ReadFile`: for code files > 150 lines: returns tree-sitter structure summary only; AI uses `ReadSymbol`, `ReadRange`, or `LspQuery` for exact details
- `ReadSymbol(file, symbol_name)`: directly extracts one named function/struct/class/impl via tree-sitter
- `ReadRange(file, start_line, end_line)`: exact local snippet extraction for surrounding scope and non-symbol context
- `WriteFile`: writes/creates file; project root boundary check; creates parent dirs; result: `"written: <path> (<N> bytes)"`; tree-sitter syntax check post-write (warnings only)
- `EditFile`: search-replace edit (old_string must match exactly once); result: `"edited: <path>"`; tree-sitter syntax check post-edit
- `ListFiles`: file paths only (gitignore-aware via `ignore` crate); no symbol summaries in output
- `BashSandbox`: allowlist check → `tokio::process::Command` with `cwd=project_root` + timeout
- `FetchUrl`: `reqwest` + domain allowlist + SSRF guard (block RFC-1918/loopback IPs)
- `create_plan(content)`: structured tool the AI calls to submit a plan (replaces fragile text parsing)
- `mark_todo_done(order_index)`: AI marks todos complete by stable order index

### Phase 7 — AI provider + core session loop (`cc-ai`)
- `AiProvider` trait + `StreamEvent` enum + `Message` / `ToolDefinition` types
- OpenAI-compatible provider (SSE streaming via `eventsource-stream`)
  - Covers: OpenAI, Ollama, LM Studio, llama.cpp (same `/v1/chat/completions` API)
- Wire tool dispatch into session loop via `cc-ai/src/tool_use.rs`
- Buffer `ToolCallInputDelta` events per call ID until `ToolCallEnd` before dispatching
- Minimal context assembly for MVP: system prompt + current request + recent turns + plan/todo state + explicit tool results
- `session.rs` REPL loop (no TUI yet) to validate the core coding loop end to end

### Phase 8 — Plan/Todo workflow
- Session state machine (`Idle → Planning → PlanPending → Executing`)
- AI calls `create_plan(content)` tool → plan persisted to DB, state → `PlanPending`
- Parse plan content into `Todo` rows (numbered/bulleted list) at plan creation time
- `/approve` command → sets `Plan::approved = true`, injects todo list (with order indices) into context, state → `Executing`
- `mark_todo_done(order_index)` tool updates todo status and refreshes plan panel

### Phase 9 — TUI (`ratatui`)
- `events.rs`: crossterm raw mode → `AppEvent` mpsc channel (bounded, capacity 64)
- `layout.rs`: 3-column + bottom bar using `ratatui::layout::Layout`
- `chat.rs`: scrollable message list, streaming text rendered via `StreamEvent` channel
- `plan.rs`: plan content + todo checklist, `/approve` keybind
- `file_tree.rs`: gitignore-aware tree via `ignore` crate, keyboard-navigable
- `input.rs`: `ratatui-textarea` multi-line input with history (up arrow)
- Token usage and session state in bottom bar

### Phase 10 — Context manipulation (`cc-ai` + `cc-context`)
- Add `symbols`, `symbol_edges`, and `diagnostics` tables plus migrations
- Incremental symbol/diagnostic index refresh hooks after file changes
- `TokenCounter` trait: `tiktoken-rs` impl for OpenAI models; `bytes/4` estimate impl for local models
- `ToolOutputCompressor` (`cc-ai/src/compressor.rs`): serializes selected artifacts into a compact IR; no blind field-dropping for structured outputs
- `RequestContextPlanner`: assembles per-turn context from plan state + current request + selected symbols/ranges + LSP facts + capped FastContext hints
- History summarization: summarize old tool chatter separately from request-scoped code context
- Context preview panel: show token breakdown and exact selected artifacts per turn

### Phase 11 — Additional providers
- Anthropic: Messages API with SSE streaming (different schema from OpenAI)
- Gemini: multipart content schema, extra mapping in provider
- Provider selection from `config.ai.default_provider`

### Phase 12 — Cross-project learning
- Add SQLite tables for reusable patterns, implementation recipes, and decision memory across projects and languages
- Learn from approved edits, completed plans, and repeated code structures
- Retrieve prior patterns when starting a new project or implementing a familiar construct such as a handler, service, or test setup
- Keep learning user-specific and inspectable: every learned pattern can be viewed, edited, or deleted

### Phase 13 — Polish
- Session summary written to DB on clean exit (via cheap fast-model call)
- Streaming recovery on partial response errors
- Comprehensive `thiserror` error chains surfaced as non-panicking TUI notifications
- Config hot-reload: watch `~/.config/contextcoder/config.toml` for changes without restart

---

## Tool Sandbox Rules

In MVP, tools enforce **boundaries only**. Milestone 2 adds `ToolOutputCompressor` for selected artifacts, but tools still should not own global context policy.

| Tool | Boundary enforced by tool |
|---|---|
| `read_file`, `read_symbol`, `write_file`, `edit_file`, `list_files` | Path must canonicalize within project root |
| `read_range` | Requested lines must stay within file bounds and project root |
| `edit_file` | `old_string` must match exactly once |
| `bash_sandbox` | First token in `allowed_bash_commands`; `cwd` = project root; no outbound network; timeout |
| `fetch_url` | Host in `fetch_allowlist`; RFC-1918/loopback blocked; no redirect bypass |
| `lsp_query` | Language server must be initialized for the detected language |
| Language support install | Requires explicit user approval before any installer command runs |
| `create_plan` | Only callable in `Planning` state |
| `mark_todo_done` | Only callable in `Executing` state |
| All tools | `curl`, `wget`, `rm -rf` NOT on default bash allowlist |

---

## Verification

End-to-end test sequence:
1. `cargo run -- init` in a test Rust project → creates `.contextcoder/` with default MD files + MVP `context.db`
2. `cargo run` → TUI launches, file tree shows project files, bottom bar shows provider + model
3. Open a TypeScript project with missing support → app asks whether it may install the TypeScript LSP and Tree-sitter grammar
4. Approve install → installer runs, verification passes, request resumes
5. Type a coding task → AI enters Planning state, produces a plan
6. `/approve` → todos appear in plan panel, state transitions to Executing
7. AI calls `read_file` + `bash_sandbox` (`cargo build`) → output shown in chat
8. AI calls `mark_todo_done(order_index)` → checklist updates in real time
9. Edit `.contextcoder/project.md` externally → context updates live (notify watcher)
10. MVP is usable for read/edit/build tasks before any advanced context manipulation is added

Unit tests:
- Round-trip TOML config serialize/deserialize, including provider-level model overriding `[ai].default_model`
- DB migration + CRUD for MVP tables (sessions, plans, todos, decisions)
- `spawn_blocking` wrapper: DB ops don't block the tokio executor (integration test with `tokio::time::timeout`)
- `BashSandbox` allowlist enforcement (blocked command returns `ToolOutput::Error`)
- `WriteFile` / `EditFile` path boundary check (path outside project root returns `ToolError`)
- `FetchUrl` SSRF guard (localhost URL returns error); allowlist mismatch returns error
- LSP bootstrap fails closed for supported languages when no server is available
- Install approval flow: deny path leaves state unchanged and surfaces remediation
- Install approval flow: approve path runs installer, verifies support, and resumes the blocked request
- Dynamic language support install/check path works for at least one Rust and one TypeScript fixture
- Tree-sitter query against a fixture `.rs` file returns correct node text
- `ToolCallInputDelta` buffering: fragments assembled correctly before dispatch

Milestone 2 tests:
- DB migration adds `symbols`, `symbol_edges`, and `diagnostics` tables without breaking MVP data
- `RequestContextPlanner` returns bounded request-scoped context for edit/debug/explain requests
- After an edit, the next planned context reflects refreshed Tree-sitter and LSP state
