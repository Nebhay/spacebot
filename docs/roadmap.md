# MVP Roadmap

Tracking progress toward a working Spacebot that can hold a conversation, delegate work, manage memory, and connect to at least one messaging platform.

For each piece: reference IronClaw, OpenClaw, Nanobot, and Rig for inspiration, but make design decisions that align with Spacebot's architecture. Don't copy patterns that assume a monolithic session model.

---

## Current State

**What exists and compiles:**
- Project structure — all modules declared, module root pattern (`src/memory.rs` not `mod.rs`)
- Error hierarchy — thiserror domain enums (`ConfigError`, `DbError`, `LlmError`, `MemoryError`, `AgentError`, `SecretsError`) wrapped by top-level `Error` with `#[from]`
- Config — hierarchical TOML config with `Config` (instance-level), `AgentConfig` (per-agent overrides), `ResolvedAgentConfig` (merged), `Binding` (messaging routing), `MessagingConfig`. Supports `env:` prefix for secret references. Falls back to env-only loading when no config file exists.
- Multi-agent — `AgentId` type, `Agent` struct (bundles db + deps + identity + prompts per agent), `agent_id` on all `ProcessEvent` variants, `AgentDeps`, per-agent database isolation. `main.rs` initializes each agent independently with its own Db, MemorySearch, event bus, and ToolServer.
- Database connections — SQLite (sqlx) + LanceDB + redb, per-agent. SQLite migrations for all tables (memories, associations, conversations, heartbeats). Migration runner in `db.rs`.
- LLM — `SpacebotModel` implements Rig's `CompletionModel` trait. Routes through `LlmManager` via direct HTTP to Anthropic and OpenAI. Handles tool definitions in requests and tool calls in responses.
- Memory — types, SQLite store (full CRUD + associations), LanceDB embedding storage + vector search + FTS, fastembed (all-MiniLM-L6-v2, 384 dims, shared via `Arc`), hybrid search (vector + FTS + graph traversal + RRF fusion), `MemorySearch` bundles store + lance + embedder. Maintenance (decay/prune stubs).
- Identity — `Identity` struct loads SOUL.md, IDENTITY.md, USER.md from agent workspace with `render()` for prompt injection. `Prompts` struct loads with fallback chain: agent workspace override → shared prompts dir → relative prompts/.
- Agent structs — Channel, Branch, Worker, Compactor, Cortex with `agent_id` threaded through via `AgentDeps`. Core LLM calls within agents are simulated.
- StatusBlock — event-driven updates from `ProcessEvent`, renders to context string
- SpacebotHook — implements `PromptHook<M>` with `agent_id`, tool call/result event emission, leak detection
- CortexHook — implements `PromptHook<M>` for system observation
- Messaging — `Messaging` trait with RPITIT + `MessagingDyn` companion + blanket impl. `MessagingManager` with adapter registry. Discord/Telegram/Webhook adapters are empty stubs.
- Tools — 11 tools implement Rig's `Tool` trait. `ReplyTool` uses `mpsc::Sender<OutboundResponse>` (ToolServer-compatible). `MemorySaveTool` includes `channel_id` field. `SetStatusTool` includes `agent_id`.
- System prompts — 5 prompt files in `prompts/` (CHANNEL.md, BRANCH.md, WORKER.md, COMPACTOR.md, CORTEX.md)
- `main.rs` — per-agent init loop (config → db → memory → identity → prompts → deps → Agent), shared LlmManager + EmbeddingModel, graceful shutdown

**What's missing:**
- Agent LLM calls are simulated (placeholder `tokio::time::sleep` instead of real `agent.prompt()`)
- ToolServer created per agent but no tools registered on it yet (Phase 4)
- No messaging routing — bindings exist in config but no Router dispatches messages to agents
- Streaming not implemented (SpacebotModel.stream() returns error)
- Secrets and settings stores are empty stubs
- No identity template files bootstrapped on agent creation

**Known issues:**
- Arrow version mismatch in Cargo.toml: `arrow = "54"` vs `arrow-array`/`arrow-schema` at `"57.3.0"` — should align or drop the `arrow` meta-crate
- `lance.rs` casts `_distance`/`_score` columns as `Float64Type` — LanceDB may return `Float32`, risking a runtime panic on cast
- `definition()` on all tools hand-writes JSON schemas instead of using the `JsonSchema` derive on `Args` types. Dual maintenance burden. Low priority cleanup.

---

## ~~Phase 1: Migrations and LanceDB~~ Done

- [x] SQLite migrations for all tables (memories, associations, conversations, heartbeats)
- [x] Inline DDL removed from `memory/store.rs`, `conversation/history.rs`, `heartbeat/store.rs`
- [x] `memory/lance.rs` — LanceDB table with Arrow schema, embedding insert, vector search (cosine), FTS (Tantivy), index creation
- [x] Embedding generation wired into memory save flow (`memory_save.rs` generates + stores)
- [x] Vector + FTS results connected into hybrid search via `MemorySearch` struct
- [x] `MemorySearch` bundles `MemoryStore` + `EmbeddingTable` + `EmbeddingModel`, replaces `memory_store` in `AgentDeps`

---

## ~~Phase 2: Wire Tools to Rig~~ Done

- [x] All 11 tools implement Rig's `Tool` trait
- [x] `AgentDeps.tool_server` uses `rig::tool::server::ToolServerHandle` directly
- [x] `PromptHook<M>` on `SpacebotHook` and `CortexHook`
- [x] `agent_id: AgentId` threaded through SpacebotHook, SetStatusTool, all ProcessEvent variants
- [x] `MemorySaveTool` — `channel_id` field added to `MemorySaveArgs`, wired into `Memory::with_channel_id()`
- [x] `ReplyTool` — replaced `Arc<InboundMessage>` with `mpsc::Sender<OutboundResponse>` for ToolServer compatibility
- [x] `EmbeddingModel` — fixed `embed_one()` to share model via `Arc` instead of creating new instance per call
- [ ] Create shared ToolServer for channel/branch tools (deferred to Phase 4)
- [ ] Create per-worker ToolServer factory for task tools (deferred to Phase 4)

---

## ~~Phase 3: System Prompts, Identity, and Multi-Agent~~ Done

- [x] `prompts/` directory with all 5 prompt files (CHANNEL.md, BRANCH.md, WORKER.md, COMPACTOR.md, CORTEX.md)
- [x] `identity/files.rs` — `Identity` struct (SOUL.md, IDENTITY.md, USER.md), `Prompts` struct with workspace-aware fallback loading
- [x] `conversation/context.rs` — `build_channel_context()`, `build_branch_context()`, `build_worker_context()`
- [x] `conversation/history.rs` — `HistoryStore` with save_turn, load_recent, compaction summaries
- [x] Multi-agent config — hierarchical TOML with `AgentConfig`, `DefaultsConfig`, `ResolvedAgentConfig`, `Binding`, `MessagingConfig`
- [x] Per-agent database isolation — each agent gets its own SQLite, LanceDB, redb in `agents/{id}/data/`
- [x] `Agent` struct bundles db + deps + identity + prompts per agent
- [x] `main.rs` per-agent initialization loop with shared LlmManager + EmbeddingModel
- [x] Prompt resolution fallback chain (agent workspace → shared prompts → relative)

---

## Phase 4: Model Routing + The Channel (MVP Core)

Implement model routing so each process type uses the right model, then wire the channel as the first real agent.

- [ ] Implement `RoutingConfig` — process-type defaults, task-type overrides, fallback chains (see `docs/routing.md`)
- [ ] Add `resolve_for_process(process_type, task_type)` to `LlmManager`
- [ ] Implement fallback logic in `SpacebotModel` — retry with next model in chain on 429/502/503/504
- [ ] Rate limit tracking — deprioritize 429'd models for configurable cooldown
- [ ] Create shared ToolServer for channel tools (reply, branch, spawn_worker, memory_save, route, cancel)
- [ ] Create per-worker ToolServer factory (shell, file, exec, set_status)
- [ ] Wire `AgentBuilder::new(model).preamble(&prompt).hook(spacebot_hook).tool_server_handle(tools).default_max_turns(5).build()`
- [ ] Replace placeholder message handling with `agent.prompt(&message).with_history(&mut history).max_turns(5).await`
- [ ] Wire status block injection — prepend rendered status to each prompt call
- [ ] Connect conversation history persistence (HistoryStore already implemented) to channel message flow
- [ ] Fire-and-forget DB writes for message persistence (`tokio::spawn`, don't block the response)
- [ ] Test: send a message to a channel, get a real LLM response back

**Reference:** `docs/routing.md` for the full routing design. Rig's `agent.prompt().with_history(&mut history).max_turns(5)` is the core call. The channel never blocks on branches, workers, or compaction.

---

## Phase 5: Branches and Workers

Replace simulated branch/worker execution with real agent calls.

- [ ] Branch: wire `agent.prompt(&task).with_history(&mut branch_history).max_turns(10).await`
- [ ] Branch result injection — insert conclusion into channel history as a distinct message
- [ ] Branch concurrency limit enforcement (already scaffolded, needs testing)
- [ ] Worker: resolve model via `resolve_for_process(Worker, Some(task_type))`, wire `agent.prompt(&task).max_turns(50).await` with task-specific tools
- [ ] Interactive worker follow-ups — repeated `.prompt()` calls with accumulated history
- [ ] Worker status reporting via set_status tool → StatusBlock updates
- [ ] Handle stale branch results and worker timeout via Rig's `MaxTurnsError` / `PromptCancelled`

**Reference:** No existing codebase has context forking. Branch is `channel_history.clone()` run independently. Workers get fresh history + task description. Rig returns chat history in error types for recovery.

---

## Phase 6: Compactor

Wire the compaction workers to do real summarization.

- [ ] Implement compaction worker — summarize old turns + extract memories via LLM
- [ ] Emergency truncation — drop oldest turns without LLM, keep N recent
- [ ] Pre-compaction archiving — write raw transcript to conversation_archives table
- [ ] Non-blocking swap — replace old turns with summary while channel continues

**Reference:** IronClaw's tiered compaction (80/85/95 thresholds, already implemented). The novel part is the non-blocking swap.

---

## Phase 7: Webhook Messaging Adapter

Get a real end-to-end messaging path working.

- [ ] Implement WebhookAdapter (axum) — POST endpoint, InboundMessage production, response routing
- [ ] Implement MessagingManager.start() — spawn adapters, merge inbound streams via `select_all`
- [ ] Implement Router — resolve InboundMessage → AgentId via bindings, dispatch to correct agent
- [ ] Implement outbound routing — responses flow from channel → manager → correct adapter
- [ ] Optional sync mode (`"wait": true` blocks until agent responds)
- [ ] Wire the full path: HTTP POST → InboundMessage → Router → Agent → Channel → response → HTTP response
- [ ] Test: curl a message in, get a response back

**Reference:** IronClaw's Channel trait and ChannelManager with `futures::stream::select_all()`. The Messaging trait and MessagingDyn companion are already implemented. See `docs/messaging.md`.

---

## Phase 8: End-to-End Integration

Wire everything together into a running system.

- [ ] Event routing — per-agent ProcessEvent dispatch (cortex, status block, logging)
- [ ] Channel lifecycle — create on first message, persist across restarts, resume from DB
- [ ] Test the full loop: message in → channel → branch → worker → memory save → response out
- [ ] Graceful shutdown — broadcast signal, drain in-flight work, close DB connections per agent

---

## Post-MVP

Not blocking the first working version, but next in line.

- **Discord adapter** — serenity 0.12.x, thread-based conversations. See `docs/discord-impl.md`.
- **Streaming** — implement `SpacebotModel.stream()` with SSE parsing, wire through messaging adapters with block coalescing (see `docs/messaging.md`)
- **Cortex** — system-level observer, memory consolidation, decay management. See `docs/cortex.md`.
- **Heartbeats** — scheduled tasks with fresh channels. Circuit breaker (3 failures → disable).
- **Telegram adapter** — teloxide, long polling mode.
- **Secrets store** — AES-256-GCM encrypted credentials in redb.
- **Settings store** — redb key-value with env > DB > default resolution.
- **Memory graph traversal during recall** — walk typed edges (Updates, Contradicts, CausedBy) during search.
- **Agent CLI** — `spacebot agents list/create/delete`, identity template bootstrapping.
- **Cross-agent communication** — routing between agents, shared observations.
