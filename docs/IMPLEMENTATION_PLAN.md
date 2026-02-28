# OPC Implementation Checklist

> Aligned to [PRD.md](./PRD.md). Each checkbox maps to a concrete deliverable.
> Mark `[x]` when implemented and verified.

---

## Phase 1: Foundation Skeleton

**PRD Ref**: §1 Core Architecture, §2 Agent Hierarchy Model, §8 Project Structure

**Goal**: Monorepo setup, type system, config parsing, agent registry, CLI scaffolding.

### 1.1 Monorepo Setup

- [ ] Initialize pnpm workspace (`pnpm-workspace.yaml`)
- [ ] Configure Turborepo (`turbo.json`) with build/test/lint pipelines
- [ ] Create shared `tsconfig.base.json`
- [ ] Configure `tsup` as build tool for library packages
- [ ] Configure Vitest as test runner
- [ ] Create package scaffolds: `@opc/core`, `@opc/cli`

### 1.2 Type System (`@opc/core/types.ts`)

- [ ] `AgentDefinition` — role, level, parent, children, model, tools, prompt path
- [ ] `MemoryBlockDefinition` — label, description, value, limit
- [ ] `MemoryConfig` — blocks, recall, archival, shareWith, sharedBlocks
- [ ] `ExecutionConfig` — maxConcurrentAgents, maxHierarchyDepth, taskTimeout, resultCompressionThreshold
- [ ] `FrameworkConfig` — name, defaultModel, memory, execution, privacy
- [ ] `ChannelConfig` — adapter reference per platform
- [ ] `SharedBlockDefinition` — description, value, limit, readOnly
- [ ] `TaskContext` — taskId, parentTaskId, agentId, description, status
- [ ] `MemoryProvenance` — author, taskId, threadId, timestamp, parentTaskId, source, contentHash
- [ ] `PIIType` enum — PERSON_NAME, EMAIL, PHONE, CREDIT_CARD, API_KEY, ADDRESS, SSN, IP_ADDRESS
- [ ] `EscalationResult` — type (error|timeout|uncertainty|approval_needed), reason, partialResult

### 1.3 Config Parser (`@opc/core/config.ts`)

- [ ] YAML loader using `yaml` package
- [ ] Zod schema: `frameworkSchema` (name, defaultModel, memory, execution, privacy)
- [ ] Zod schema: `agentSchema` (role, level, parent, children, tools, model, memory, prompt)
- [ ] Zod schema: `channelSchema` (adapter reference)
- [ ] Zod schema: `sharedBlockSchema` (description, value, limit, readOnly)
- [ ] Zod schema: `privacyConfigSchema` (enabled, detectTypes, whitelist, confidenceThreshold, mode)
- [ ] Top-level `opcConfigSchema` composing all sub-schemas
- [ ] `loadConfig(path)` — reads YAML, parses, validates, returns typed config
- [ ] Clear error messages on validation failure (path to invalid field + reason)

### 1.4 Agent Registry (`@opc/core/registry.ts`)

- [ ] `AgentRegistry` class: parse agent definitions from config
- [ ] Build adjacency graph (parent-child relationships)
- [ ] Validate: no cycles (DFS-based cycle detection)
- [ ] Validate: no orphans (all agents reachable from root)
- [ ] Validate: `shareWith` only between agents at same level
- [ ] Validate: leaf agents (no children) do NOT have `delegate`/`delegateParallel` tools
- [ ] Validate: `sharedBlocks` references exist in top-level `sharedBlocks` definition
- [ ] Validate: `maxHierarchyDepth` not exceeded
- [ ] Validate: exactly one root agent (level 0)
- [ ] `getAgent(id)` — returns AgentDefinition
- [ ] `getChildren(id)` — returns child agent IDs
- [ ] `getParent(id)` — returns parent agent ID
- [ ] `getAncestors(id)` — returns ancestor chain
- [ ] `isDescendant(agentId, ancestorId)` — hierarchy check

### 1.5 CLI (`@opc/cli`)

- [ ] CLI entry point using `commander`
- [ ] `opc init` — scaffold `opc.config.yaml` + `agents/` directory with example prompts
- [ ] `opc validate` — load config → run all registry validations → report errors
- [ ] `opc visualize` — print agent hierarchy as ASCII tree in terminal

### 1.6 Tests — Phase 1

- [ ] `config.test.ts` — valid configs pass, invalid configs produce clear errors
- [ ] `config.test.ts` — edge cases: missing fields, extra fields, wrong types
- [ ] `registry.test.ts` — cycle detection catches circular parent-child refs
- [ ] `registry.test.ts` — orphan detection catches unreachable agents
- [ ] `registry.test.ts` — shareWith validation (same level only)
- [ ] `registry.test.ts` — leaf-no-delegate validation
- [ ] `registry.test.ts` — sharedBlocks reference validation
- [ ] `registry.test.ts` — depth limit validation
- [ ] CLI integration: `opc init` creates expected files
- [ ] CLI integration: `opc validate` catches all error types

---

## Phase 2: Privacy Gateway

**PRD Ref**: §4 Privacy Gateway

**Goal**: PII detection, masking, restoration pipeline. Transparent to agents.

**Depends on**: Phase 1 (types)

### 2.1 Types (`@opc/privacy/types.ts`)

- [ ] `PIIType` enum (PERSON_NAME, EMAIL, PHONE, CREDIT_CARD, API_KEY, ADDRESS, SSN, IP_ADDRESS, JWT, SSH_KEY)
- [ ] `DetectionResult` — value, type, confidence, startIndex, endIndex
- [ ] `VaultEntry` — realValue, fakeValue, entityType
- [ ] `PrivacyConfig` — enabled, detectTypes, whitelist, confidenceThreshold, mode (mask|redact|route)
- [ ] `MaskMode` type: `'mask' | 'redact' | 'route'`

### 2.2 PII Detector (`@opc/privacy/detector.ts`)

- [ ] `detectStructured(text)` — regex-based detection using OpenRedaction (570+ patterns)
- [ ] Structured detection: email, phone, credit card (Luhn), API key, JWT, SSH key, IP address, SSN
- [ ] `detectUnstructured(text)` — NER-based detection via transformers.js (BERT ONNX)
- [ ] NER detection: person names, organization names, addresses
- [ ] Lazy model loading: NER model (~400MB) downloaded on first call, cached for subsequent use
- [ ] `detect(text)` — merge structured + unstructured, deduplicate overlapping detections
- [ ] Confidence threshold filtering (configurable, default 0.85)
- [ ] Whitelist filtering: skip detections matching whitelisted terms

### 2.3 Substitutor (`@opc/privacy/substitutor.ts`)

- [ ] `generate(type, seed)` — type-preserving fake data via Faker.js
- [ ] Type preservation: name→name, email→email, phone→phone, address→address, etc.
- [ ] Seeded Faker instance: `faker.seed()` for session-level consistency

### 2.4 Vault (`@opc/privacy/vault.ts`)

- [ ] `PrivacyVault` class with session-scoped `Map<real, fake>` + `Map<fake, real>`
- [ ] `getOrCreate(realValue, type)` — idempotent: same real value always returns same fake
- [ ] `restore(maskedText)` — longest-match-first restoration (sort by key length descending)
- [ ] `clear()` — session cleanup, vault not persisted
- [ ] `entityTypes` map: track `real → PIIType` for each mapping

### 2.5 Gateway (`@opc/privacy/gateway.ts`)

- [ ] `mask(text)` — detect → substitute → return masked text
- [ ] `unmask(text)` — vault.restore()
- [ ] `wrapQuery(queryFn)` — higher-order function wrapping agent `query()` calls
- [ ] Whitelist pre-filtering before substitution
- [ ] Code block conservative mode: only replace obvious keys/connection strings within fenced code blocks
- [ ] Support for `redact` mode: replace PII with `[REDACTED]` instead of fake data

### 2.6 Stream Handler (`@opc/privacy/stream-handler.ts`)

- [ ] `StreamRestorer` class: pipe streaming response through vault restoration
- [ ] Buffer chunks to token boundaries before replacement
- [ ] Handle partial fake-data tokens spanning chunk boundaries

### 2.7 SDK Integration (`@opc/privacy/hooks.ts`)

- [ ] `privacyMaskHook` — PreToolUse hook for Claude Agent SDK
- [ ] `privacyUnmaskHook` — PostToolUse hook for Claude Agent SDK
- [ ] Transparent injection: agents are unaware of masking

### 2.8 Tests — Phase 2

- [ ] `detector.test.ts` — detection accuracy per PII type (email, phone, CC, API key, names, etc.)
- [ ] `detector.test.ts` — confidence threshold filtering works
- [ ] `detector.test.ts` — whitelist exclusion works
- [ ] `vault.test.ts` — bidirectional mapping consistency within session
- [ ] `vault.test.ts` — longest-match-first restoration (e.g., "Zhang Sanfeng" before "Zhang San")
- [ ] `gateway.test.ts` — round-trip: mask → unmask produces original text exactly
- [ ] `gateway.test.ts` — code block conservative strategy
- [ ] `gateway.test.ts` — redact mode replaces with `[REDACTED]`
- [ ] `stream-handler.test.ts` — streaming restoration across chunk boundaries

---

## Phase 3: Memory System

**PRD Ref**: §5 Memory System (§4.1–§4.5 in PRD subsections)

**Goal**: Core Memory blocks, Recall Memory (dual-write + hybrid search), ACL, provenance.

**Depends on**: Phase 1 (types, config)

### 3.1 Storage Adapter Interface (`@opc/memory/adapters/adapter.ts`)

- [ ] `StorageAdapter` interface: CRUD for blocks, recall entries, semantic facts, provenance
- [ ] Method signatures for vector search, BM25 search, batch operations

### 3.2 SQLite Adapter (`@opc/memory/adapters/sqlite.ts`)

- [ ] `SQLiteAdapter` class using `better-sqlite3`
- [ ] Schema: `memory_blocks` table (id, label, description, value, limit, readOnly, version, agentId)
- [ ] Schema: `recall_messages` table (id, agentId, role, content, threadId, timestamp, contentHash, processed)
- [ ] Schema: `recall_vectors` virtual table (sqlite-vec for cosine similarity)
- [ ] Schema: `recall_fts` virtual table (FTS5 for BM25)
- [ ] Schema: `semantic_facts` table (id, agentId, fact, validTime, recordedTime, contentHash, namespace)
- [ ] Schema: `semantic_vectors` virtual table (sqlite-vec)
- [ ] Schema: `provenance` table (entryId, entryType, author, taskId, threadId, timestamp, parentTaskId, source, contentHash)
- [ ] Migration runner for schema versioning
- [ ] Zero external dependencies (SQLite + extensions only)

### 3.3 Embedding Provider (`@opc/memory/embedding.ts`)

- [ ] `EmbeddingProvider` interface: `embed(text): Promise<number[]>`, `embedBatch(texts): Promise<number[][]>`
- [ ] `LocalEmbedding` — transformers.js + all-MiniLM-L6-v2 ONNX (~100MB, auto-download on first use)
- [ ] `OpenAIEmbedding` — OpenAI API (text-embedding-3-small, configurable dimensions)
- [ ] `VoyageEmbedding` — Voyage API
- [ ] `createEmbeddingProvider(config)` — factory function based on config

### 3.4 Core Memory Blocks (`@opc/memory/blocks.ts`)

**PRD Ref**: §4.1 Core Memory — Memory Blocks

- [ ] `MemoryBlockStore` class
- [ ] `get(agentId, label)` — retrieve block by agent + label
- [ ] `set(agentId, label, value)` — create or update block
- [ ] Optimistic locking via `version` column (reject if version mismatch)
- [ ] `readOnly` enforcement: reject writes to readOnly blocks
- [ ] Character limit validation: `value.length <= limit`
- [ ] Shared block support: blocks with `agentId = null` referenced by multiple agents
- [ ] Shared block write: validate `!readOnly` before allowing updates
- [ ] Blocks always prepended to system prompt (handled by ContextBuilder)

### 3.5 Recall Memory (`@opc/memory/recall.ts`)

**PRD Ref**: §4.2 Recall Memory — Conversation History

- [ ] `RecallMemory` class
- [ ] `write(message)` — synchronous SQL insert + asynchronous vector embedding (non-blocking)
- [ ] Role-based embedding format:
  - [ ] `user` → `{"content": "user message text"}`
  - [ ] `assistant + tool_call` → `{"thinking": "CoT", "content": "reply"}`
  - [ ] `tool result` → `{"tool_call": "...", "tool_result": "..."}`
  - [ ] `system` → skipped (not embedded)
- [ ] Content hash dedup: MD5 hash to prevent duplicate entries
- [ ] Semantic threshold dedup (cosine similarity > 0.7 = duplicate)

### 3.6 Hybrid Search (`@opc/memory/hybrid-search.ts`)

**PRD Ref**: §4.2 Retrieval — Hybrid strategy

- [ ] `vectorSearch(query, k)` — sqlite-vec cosine similarity
- [ ] `bm25Search(query, k)` — FTS5 BM25 scoring
- [ ] `fuse(vectorResults, bm25Results, k)` — Reciprocal Rank Fusion (RRF, k=60)
- [ ] RRF formula: `score = sum(1 / (k + rank_i))`
- [ ] Configurable weight split (default: 70% vector, 30% BM25 via RRF)

### 3.7 Namespace ACL (`@opc/memory/acl.ts`)

**PRD Ref**: §4.4 Hierarchical Access Control

- [ ] `MemoryACL` class
- [ ] Namespace structure: `/org/agent/{agentId}/{tier}` (core, recall, semantic)
- [ ] Shared namespace: `/org/shared/{blockLabel}`
- [ ] Permission: own namespace — full read/write
- [ ] Permission: child namespace — read + write (superior reads/directs subordinate)
- [ ] Permission: parent namespace — no access
- [ ] Permission: sibling namespace — configurable via `shareWith`
- [ ] Permission: shared blocks — read if referenced, write if `!readOnly`
- [ ] `checkPermission(requesterAgentId, targetAgentId, operation, tier)` → boolean
- [ ] Enforce ACL as middleware wrapping all memory reads/writes

### 3.8 Provenance Tracking (`@opc/memory/provenance.ts`)

**PRD Ref**: §4.5 Provenance Tracking

- [ ] `ProvenanceTracker` class
- [ ] `attach(entryId, entryType, metadata)` — attach provenance to any memory entry
- [ ] Provenance fields: author, taskId, threadId, timestamp, parentTaskId, source (agent|consolidation|human|system), contentHash
- [ ] `query(filters)` — search provenance by author, taskId, threadId, time range
- [ ] Content hash generation (MD5)

### 3.9 Memory System Facade (`@opc/memory/index.ts`)

- [ ] `MemorySystem` class: unified init, getBlockStore, getRecall, getACL, getProvenance
- [ ] Config-driven initialization (SQLite by default)

### 3.10 Tests — Phase 3

- [ ] `blocks.test.ts` — CRUD operations
- [ ] `blocks.test.ts` — optimistic lock conflict detection
- [ ] `blocks.test.ts` — readOnly enforcement
- [ ] `blocks.test.ts` — shared block reads + writes
- [ ] `blocks.test.ts` — character limit validation
- [ ] `recall.test.ts` — dual-write (SQL + vector) correctness
- [ ] `recall.test.ts` — hybrid search returns relevant results
- [ ] `recall.test.ts` — content hash dedup
- [ ] `recall.test.ts` — role-based embedding format
- [ ] `hybrid-search.test.ts` — RRF fusion ranking quality
- [ ] `hybrid-search.test.ts` — combined outperforms either alone on synthetic queries
- [ ] `acl.test.ts` — self read/write: allowed
- [ ] `acl.test.ts` — child read/write: allowed
- [ ] `acl.test.ts` — parent read/write: denied
- [ ] `acl.test.ts` — sibling without shareWith: denied
- [ ] `acl.test.ts` — sibling with shareWith: allowed
- [ ] `acl.test.ts` — shared block read (referenced): allowed
- [ ] `acl.test.ts` — shared block write (!readOnly): allowed
- [ ] `acl.test.ts` — shared block write (readOnly): denied
- [ ] `provenance.test.ts` — metadata attachment + querying
- [ ] `sqlite.test.ts` — schema creation, migrations, all CRUD operations

---

## Phase 4: Orchestration Engine

**PRD Ref**: §1 Core Architecture (N-Layer), §5 Context Engineering, §6 Message Routing

**Goal**: delegate tool, multi-layer orchestration, context assembly, event bus.

**Depends on**: Phase 1 (types, registry), Phase 3 (memory)

### 4.1 Task Router (`@opc/core/router.ts`)

- [ ] `TaskRouter` class
- [ ] `route(parentAgentId, targetAgentId)` — validate parent→child relationship
- [ ] Reject: delegation to non-child (sibling, grandchild, ancestor)
- [ ] Uses registry adjacency graph

### 4.2 Orchestrator (`@opc/core/orchestrator.ts`)

**PRD Ref**: §1 Recursive query() calls, §6 Task Lifecycle

- [ ] `Orchestrator` class
- [ ] `execute(task)` — entry point, creates root `query()` session
- [ ] `delegate(parentSession, targetAgentId, taskDescription)` — validate route → create child session → run `query()` → compress result → return
- [ ] Session tracking: `Map<sessionId, SessionState>`
- [ ] Concurrency control: semaphore limiting `maxConcurrentAgents`
- [ ] Depth tracking: reject if depth > `maxHierarchyDepth`
- [ ] Timeout enforcement per task (`taskTimeout` config)
- [ ] Task lifecycle states: `CREATED → ASSIGNED → DELEGATED → EXECUTING → RESULT_PROPAGATING → COMPLETED/FAILED`
- [ ] Result compression: if child result > `resultCompressionThreshold` tokens, summarize with fast model (Haiku)

### 4.3 Context Builder (`@opc/core/context-builder.ts`)

**PRD Ref**: §5 Context Engineering — Write/Select/Compress/Isolate

- [ ] `ContextBuilder` class
- [ ] **Write** (static injection):
  - [ ] System prompt from `agents/{id}/prompt.md`
  - [ ] Core Memory blocks prepended (always visible)
  - [ ] Org structure injection: "You are {role}, your superior is {parent.role}, your subordinates are {children}"
- [ ] **Select** (dynamic retrieval):
  - [ ] Recall Memory: hybrid search using task description, top-5
  - [ ] Semantic Memory: relevant fact retrieval
  - [ ] Parent task instructions: specific task from `delegate` call
- [ ] **Compress** (window fitting):
  - [ ] Core blocks retained in full (constrained by per-block limits)
  - [ ] Recall results summarized if exceeding token budget
  - [ ] Only pass direct parent instructions, NOT full ancestor chain
- [ ] **Isolate** (security boundaries):
  - [ ] Each `query()` has independent context window
  - [ ] Sibling agents invisible to each other (unless `shareWith`)
  - [ ] Child results > 2000 tokens summarized before returning to parent

### 4.4 Event Bus (`@opc/core/event-bus.ts`)

- [ ] `EventBus` class (Node.js EventEmitter-based)
- [ ] Event types: `SubagentStart`, `SubagentStop`, `TaskDelegated`, `TaskCompleted`, `TaskFailed`, `MemoryUpdated`
- [ ] `emit(event)` — publish to subscribers
- [ ] `on(eventType, handler)` — subscribe
- [ ] Hook into orchestrator lifecycle (start/stop/delegate/complete/fail)

### 4.5 Tools (`@opc/tools`)

**PRD Ref**: §1 delegate MCP tool, §4.1 memory self-edit tools, §6 escalation

- [ ] `delegate` MCP tool — input: targetAgentId + taskDescription → validate route → orchestrator.delegate() → return compressed result
- [ ] `delegateParallel` MCP tool — input: Array<{agentId, task}> → Promise.allSettled on multiple delegate calls → aggregated results
- [ ] `memory_replace(block_label, old_text, new_text)` — precise replacement in Core Memory block
- [ ] `memory_insert(block_label, new_text)` — append content to Core Memory block
- [ ] `memory_read(query)` — search Recall + Semantic memory, ACL-filtered
- [ ] `escalate(type, reason, partialResult)` — structured return to parent (error|timeout|uncertainty|approval_needed)
- [ ] `human_approval(question, options)` — pause execution, wait for human response

### 4.6 Tests — Phase 4

- [ ] `router.test.ts` — valid child delegation: allowed
- [ ] `router.test.ts` — sibling/grandchild/ancestor delegation: rejected
- [ ] `orchestrator.test.ts` — single delegate call works end-to-end
- [ ] `orchestrator.test.ts` — 3-layer chain: CEO → VP → Dev completes
- [ ] `orchestrator.test.ts` — concurrency limit enforced (maxConcurrentAgents)
- [ ] `orchestrator.test.ts` — timeout kills stalled tasks
- [ ] `orchestrator.test.ts` — result compression triggers for large results
- [ ] `orchestrator.test.ts` — task lifecycle state transitions correct
- [ ] `context-builder.test.ts` — Write stage: system prompt + blocks + org structure present
- [ ] `context-builder.test.ts` — Select stage: recall + semantic results included
- [ ] `context-builder.test.ts` — Compress stage: oversized recall summarized
- [ ] `context-builder.test.ts` — Isolate stage: sibling data excluded
- [ ] `delegate.test.ts` — tool schema validates input correctly
- [ ] `delegate-parallel.test.ts` — parallel delegation with partial failures

---

## Phase 5: Semantic Memory + Consolidation

**PRD Ref**: §4.3 Semantic Memory, §4.6 Sleep-time Consolidation

**Goal**: Fact extraction pipeline, dedup, graph memory, sleep-time consolidation.

**Depends on**: Phase 3 (memory, embedding, SQLite adapter)

### 5.1 Semantic Memory (`@opc/memory/semantic.ts`)

**PRD Ref**: §4.3 Fact Extraction — Mem0-style 5-stage pipeline

- [ ] `SemanticMemory` class
- [ ] **Stage 1 — Fact Extraction**: LLM extracts atomic facts from conversations (preferences, decisions, entity info)
- [ ] **Stage 2 — Embedding**: vectorize each extracted fact
- [ ] **Stage 3 — Conflict Detection**: top-10 similarity search against existing semantic memory
- [ ] **Stage 4 — Action Decision**: LLM determines `ADD` / `UPDATE` / `DELETE` / `NOOP` per fact
- [ ] **Stage 5 — Parallel Write**: simultaneously write to Vector DB + optional Graph DB
- [ ] Dedup layer 1: content hash check
- [ ] Dedup layer 2: semantic threshold (cosine similarity > 0.7)
- [ ] Dedup layer 3: LLM judgment (ADD/UPDATE/DELETE)
- [ ] Dedup layer 4: scope-based (same agent namespace)

### 5.2 Bi-temporal Tracking

**PRD Ref**: §4.3 Time awareness (inspired by Zep)

- [ ] `valid_time` field: when the fact was true in the real world
- [ ] `recorded_time` field: when the fact was stored in memory
- [ ] Temporal queries: "what was true at time T?" using valid_time

### 5.3 Graph Memory (`@opc/memory/graph.ts`)

**PRD Ref**: §4.3 Graph Memory (optional)

- [ ] `GraphMemory` class
- [ ] `addEntity(name, type, embedding, agentId)` — create entity node
- [ ] `addRelation(source, target, type)` — create typed edge (LIVES_IN, PREFERS, DEPENDS_ON, etc.)
- [ ] `search(query)` — BM25 reranking on graph triples
- [ ] Default implementation: SQLite relational tables (`entities` + `relationships`)
- [ ] Schema: `entities` table (id, name, type, embedding, agentId, createdAt, updatedAt)
- [ ] Schema: `relationships` table (id, sourceId, targetId, type, properties, createdAt)

### 5.4 Sleep-time Consolidator (`@opc/memory/consolidator.ts`)

**PRD Ref**: §4.6 Sleep-time Consolidation

- [ ] `SleepTimeConsolidator` class
- [ ] Trigger: every N agent interactions (configurable via `consolidation.frequency`, default 5)
- [ ] Executor: independent consolidation agent using low-cost model (`consolidation.model`, default Haiku)
- [ ] Step 1: read recent unprocessed messages from agent's Recall memory
- [ ] Step 2: run Mem0-style fact extraction pipeline → write to Semantic Memory
- [ ] Step 3: reflect on current Core Memory blocks → generate update suggestions → auto-apply
- [ ] Step 4: mark processed Recall entries (`processed = true`)
- [ ] Runs as async background task (non-blocking)

### 5.5 Tests — Phase 5

- [ ] `semantic.test.ts` — fact extraction produces reasonable atomic facts from synthetic conversations
- [ ] `semantic.test.ts` — conflict resolution: ADD for new facts
- [ ] `semantic.test.ts` — conflict resolution: UPDATE for contradicting facts
- [ ] `semantic.test.ts` — conflict resolution: DELETE for outdated facts
- [ ] `semantic.test.ts` — 4-layer dedup catches duplicates at each level
- [ ] `semantic.test.ts` — bi-temporal queries return correct temporal slices
- [ ] `graph.test.ts` — entity CRUD
- [ ] `graph.test.ts` — relationship CRUD
- [ ] `graph.test.ts` — triple search returns relevant results
- [ ] `consolidator.test.ts` — trigger logic (fires after N interactions, not before)
- [ ] `consolidator.test.ts` — full consolidation flow: recall → extract → semantic + core updates

---

## Phase 6: Communication Integration

**PRD Ref**: §3 Communication Layer — Vercel Chat SDK Integration

**Goal**: Chat SDK gateway, streaming bridge, JSX cards, webhook server.

**Depends on**: Phase 4 (orchestrator, event bus)

### 6.1 Chat Gateway (`@opc/gateway/chat-gateway.ts`)

**PRD Ref**: §3 Chat SDK event handling

- [ ] `ChatGateway` class
- [ ] `onNewMention(event)` — new conversation, route to root agent
- [ ] `onSubscribedMessage(event)` — follow-up message, restore existing session
- [ ] `onAction(event)` — button interactions (approvals, cancellations)
- [ ] `onSlashCommand(event)` — handle `/status`, `/agents`, etc.
- [ ] Session restore from thread context

### 6.2 Stream Bridge (`@opc/gateway/stream-bridge.ts`)

**PRD Ref**: §3 Streaming output

- [ ] `StreamBridge` class
- [ ] `bridge(queryGenerator)` — `query()` AsyncGenerator → AsyncIterable<string>
- [ ] Backpressure handling
- [ ] Error propagation from agent to chat stream
- [ ] Integration with `thread.post()` native streaming

### 6.3 JSX Card Components (`@opc/gateway/cards/`)

- [ ] `status-card.tsx` — task status with progress indicator
- [ ] `approval-card.tsx` — approval request with Accept/Reject buttons → `onAction` handler
- [ ] `agent-tree-card.tsx` — compact agent hierarchy view

### 6.4 Webhook Server (`apps/server/`)

- [ ] Next.js API route: `slack/route.ts` — Slack webhook → ChatGateway
- [ ] Next.js API route: `discord/route.ts` — Discord webhook → ChatGateway
- [ ] Platform-specific payload parsing

### 6.5 Tests — Phase 6

- [ ] `chat-gateway.test.ts` — event routing: mention → root agent
- [ ] `chat-gateway.test.ts` — event routing: subscribed message → session restore
- [ ] `chat-gateway.test.ts` — event routing: action → handler
- [ ] `chat-gateway.test.ts` — event routing: slash command → handler
- [ ] `stream-bridge.test.ts` — AsyncGenerator to AsyncIterable conversion
- [ ] `stream-bridge.test.ts` — backpressure handling
- [ ] `stream-bridge.test.ts` — error propagation
- [ ] Integration: webhook → gateway → orchestrator → stream response

---

## Phase 7: Web UI Dashboard

**PRD Ref**: §7 Web UI Dashboard

**Goal**: Full dashboard with all pages, real-time updates.

**Depends on**: Phase 3 (memory), Phase 4 (event bus, orchestrator)

### 7.1 Project Setup (`apps/dashboard/`)

- [ ] Next.js 15 (App Router) setup
- [ ] shadcn/ui + BaseUI component library integration
- [ ] Tailwind CSS configuration
- [ ] React Flow dependency
- [ ] WebSocket client for EventBus

### 7.2 Layout & Navigation

- [ ] Root layout with sidebar navigation (`layout.tsx`)
- [ ] Sidebar component linking to all pages

### 7.3 Dashboard Overview (`page.tsx`)

**PRD Ref**: §7 Dashboard page

- [ ] Active task count display
- [ ] Agent status indicators (idle / busy / error)
- [ ] Recent activity timeline

### 7.4 Agent Tree (`agents/page.tsx`)

**PRD Ref**: §7 Agent Tree page

- [ ] React Flow interactive hierarchy graph
- [ ] Custom agent nodes (`agent-node.tsx`) showing role, status, model
- [ ] Click agent node → view details panel (config, current task, memory summary)
- [ ] Real-time status updates via WebSocket

### 7.5 Task Flow (`tasks/page.tsx`)

**PRD Ref**: §7 Task Flow page

- [ ] Real-time task delegation chain visualization
- [ ] Custom task nodes (`task-node.tsx`) showing status, timing, agent
- [ ] Show current execution position in chain
- [ ] WebSocket-driven updates (TaskDelegated, TaskCompleted, TaskFailed events)

### 7.6 Memory Viewer (`memory/page.tsx`)

**PRD Ref**: §7 Memory Viewer page

- [ ] Browse by agent selector
- [ ] Tab: Core Memory block editor (`memory-block-editor.tsx`) — view/edit blocks
- [ ] Tab: Recall Memory search (`recall-search.tsx`) — search query input + results
- [ ] Tab: Semantic Memory fact list (`semantic-list.tsx`) — filterable list
- [ ] Tab: Graph visualization (entity-relationship diagram, if graph enabled)

### 7.7 Config Editor (`config/page.tsx`)

**PRD Ref**: §7 Config Editor page

- [ ] Monaco Editor for YAML editing
- [ ] Schema validation (Zod) with inline error display
- [ ] Hierarchy preview (live React Flow mini-graph)

### 7.8 Logs (`logs/page.tsx`)

**PRD Ref**: §7 Logs page

- [ ] Aggregated logs from all agent executions
- [ ] Filter by: task ID, agent ID, time range
- [ ] Log viewer component (`log-viewer.tsx`)

### 7.9 Privacy Dashboard (`privacy/page.tsx`)

**PRD Ref**: §4 Privacy Gateway (monitoring)

- [ ] Masking statistics (count by PII type)
- [ ] Vault status (active sessions, mapping counts)
- [ ] Detection logs (recent detections with type + confidence)

### 7.10 Real-time Data Flow

**PRD Ref**: §7 Real-time Data Flow

- [ ] `use-websocket.ts` hook — WebSocket connection to EventBus
- [ ] `use-agents.ts` hook — agent data fetching + real-time updates
- [ ] `use-tasks.ts` hook — task data fetching + real-time updates
- [ ] `use-memory.ts` hook — memory data fetching
- [ ] `lib/api.ts` — API client for backend

### 7.11 Tests — Phase 7

- [ ] Component tests: each page renders with mock data
- [ ] WebSocket integration: connection receives real-time events
- [ ] E2E (optional): Playwright tests for critical flows (view agent, search memory, edit config)

---

## Phase 8: Production Hardening

**PRD Ref**: §10 Implementation Steps — Phase 8

**Goal**: Production-grade adapters, containerization, documentation, CI/CD.

**Depends on**: All previous phases

### 8.1 Postgres Adapter (`@opc/memory/adapters/postgres.ts`)

- [ ] `PostgresAdapter` class using `pg`
- [ ] pgvector extension for vector similarity search
- [ ] Native FTS for BM25 search
- [ ] All memory operations: blocks, recall, semantic, provenance
- [ ] Passes same test suite as SQLite adapter

### 8.2 Neo4j Adapter (`@opc/memory/adapters/neo4j.ts`)

- [ ] `Neo4jAdapter` class using `neo4j-driver`
- [ ] Cypher queries for entity/relationship CRUD
- [ ] Full graph traversal capabilities
- [ ] BM25 reranking on graph triples

### 8.3 Containerization

- [ ] `Dockerfile` — multi-stage build for server + dashboard
- [ ] `docker-compose.yml` — services: opc-server, dashboard, postgres, neo4j
- [ ] Docker Compose profiles: `dev` (SQLite), `prod` (Postgres + Neo4j)
- [ ] Smoke test: `docker-compose up` starts all services

### 8.4 CI/CD

- [ ] `.github/workflows/ci.yml` — lint + test + build on PR
- [ ] `.github/workflows/release.yml` — publish to npm on tag
- [ ] All tests pass in CI environment

### 8.5 Documentation

- [ ] `docs/getting-started.md` — quick start guide
- [ ] `docs/configuration.md` — full config reference
- [ ] `docs/architecture.md` — architecture overview
- [ ] `docs/deployment.md` — deployment guide (local, Docker, production)
- [ ] Example configurations (minimal, full-featured)

---

## End-to-End Verification

**PRD Ref**: §11 Verification Plan

- [ ] **Privacy Gateway E2E**: PII detection accuracy covering all types + Vault consistency + streaming restoration + whitelist exclusion + code block conservative strategy
- [ ] **ACL E2E**: Full combinatorial coverage of permission matrix (self/superior/subordinate/sibling × read/write × each tier)
- [ ] **Memory Integration E2E**: block self-edit → recall dual-write + hybrid retrieval → fact extraction (ADD/UPDATE/DELETE) → dedup → consolidation → temporal axis queries
- [ ] **Orchestration E2E**: 3-layer delegate chain (CEO → VP → Dev), message passing + result compression + parallel delegation
- [ ] **Privacy + Orchestration E2E**: User message with PII → mask → 3-layer agent processing → restore → verify real data never appears in LLM API logs
- [ ] **Pure Local E2E**: Offline environment + Ollama + SQLite, verify full pipeline runs with zero cloud dependencies
- [ ] **Dashboard E2E**: Launch → trigger task → verify real-time updates / memory viewing / config editing / privacy statistics

---

## Dependency Graph

```
Phase 1 (Foundation) ─────┬──► Phase 2 (Privacy)
                          ├──► Phase 3 (Memory) ──────┬──► Phase 5 (Semantic + Consolidation)
                          │                            │
                          └──► Phase 4 (Orchestration) ┤
                               (needs Phase 1 + 3)     ├──► Phase 6 (Communication)
                                                       │    (needs Phase 4)
                                                       │
                                                       ├──► Phase 7 (Dashboard)
                                                       │    (needs Phase 3 + 4)
                                                       │
                                                       └──► Phase 8 (Production)
                                                            (needs all)
```

**Parallelizable**: Phase 2 (Privacy) and Phase 3 (Memory) can run in parallel after Phase 1.

---

## Summary

| Phase | Checkboxes | Description |
|-------|-----------|-------------|
| 1 | 49 | Foundation: monorepo, types, config, registry, CLI |
| 2 | 30 | Privacy: PII detection, vault, mask/unmask, streaming |
| 3 | 45 | Memory: blocks, recall, hybrid search, ACL, provenance |
| 4 | 34 | Orchestration: delegate, orchestrator, context builder, events |
| 5 | 22 | Semantic: fact extraction, dedup, graph, consolidation |
| 6 | 17 | Communication: chat gateway, stream bridge, JSX cards |
| 7 | 26 | Dashboard: 7+ pages, real-time updates |
| 8 | 13 | Production: Postgres, Neo4j, Docker, docs, CI/CD |
| E2E | 7 | End-to-end verification plan |
| **Total** | **~243** | |
