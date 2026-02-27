# OPC Implementation Plan

## Overview

This document breaks down the OPC PRD into actionable implementation phases with concrete technical decisions, file-level breakdowns, dependencies, complexity estimates, and testing strategies.

---

## Phase 1: Foundation Skeleton

**Goal**: Monorepo setup, type system, config parsing, agent registry, CLI scaffolding.

**Estimated Complexity**: Low-Medium (~1 week)

### Technical Decisions
- **Monorepo tool**: pnpm workspaces + Turborepo (fast incremental builds, well-supported)
- **Schema validation**: Zod (runtime validation + TypeScript type inference)
- **YAML parsing**: `yaml` package (full YAML 1.2 spec support)
- **CLI framework**: `commander` (lightweight, widely adopted)
- **Build tool**: `tsup` (fast esbuild-based bundler for libraries)
- **Testing**: Vitest (fast, native ESM, Vite-compatible)

### File Breakdown

```
opc/
  package.json                     # Root workspace config
  pnpm-workspace.yaml             # Workspace packages definition
  turbo.json                       # Turborepo pipeline config
  tsconfig.base.json              # Shared TS config

  packages/
    core/
      package.json                 # @opc/core
      tsconfig.json
      tsup.config.ts
      src/
        types.ts                   # AgentDefinition, MemoryBlock, TaskContext, Provenance,
                                   # MemoryConfig, ExecutionConfig, PIIType, etc.
        config.ts                  # YAML loader + Zod schema (frameworkSchema, agentSchema,
                                   # channelSchema, sharedBlockSchema)
        registry.ts                # AgentRegistry class: parse topology, build adjacency graph,
                                   # validate (no cycles via DFS, no orphans, shareWith level check,
                                   # leaf-no-delegate, sharedBlocks existence)
        index.ts                   # Public API exports
      __tests__/
        config.test.ts             # Schema validation: valid/invalid configs, edge cases
        registry.test.ts           # Topology validation: cycles, orphans, shareWith, depth limits

    cli/
      package.json                 # @opc/cli
      tsconfig.json
      src/
        index.ts                   # CLI entry point (commander setup)
        commands/
          init.ts                  # Scaffold opc.config.yaml + agents/ directory
          validate.ts              # Load config, run registry validation, report errors
          visualize.ts             # Print agent tree to terminal (tree-like ASCII output)
```

### Dependencies Between Files
- `cli/commands/*` -> `core/config.ts` -> `core/types.ts`
- `cli/commands/validate.ts` -> `core/registry.ts`

### Testing Strategy
- Unit tests for Zod schema: valid configs pass, invalid configs produce clear errors
- Unit tests for AgentRegistry: cycle detection, orphan detection, shareWith validation, depth limits
- CLI integration tests: `opc init` creates expected files, `opc validate` catches errors

### Exit Criteria
- [ ] `pnpm install` works across all packages
- [ ] `opc init` generates a valid starter config
- [ ] `opc validate` catches all topology errors (cycles, orphans, invalid refs)
- [ ] All types exported and usable from `@opc/core`

---

## Phase 2: Privacy Gateway

**Goal**: PII detection, masking, and restoration pipeline that transparently wraps LLM calls.

**Estimated Complexity**: Medium-High (~1.5 weeks)

**Depends on**: Phase 1 (types)

### Technical Decisions
- **Regex PII detection**: `open-redaction` (570+ patterns, maintained)
- **NER PII detection**: `@nicepkg/pii-paladin` or equivalent transformers.js-based NER (BERT ONNX, ~400MB, downloaded on demand)
- **Fake data generation**: `@faker-js/faker` (comprehensive, seeded for consistency)
- **Streaming buffer**: Custom implementation using TextDecoder chunking at token boundaries

### File Breakdown

```
packages/
  privacy/
    package.json                   # @opc/privacy
    tsconfig.json
    tsup.config.ts
    src/
      types.ts                     # PIIType enum, DetectionResult, VaultEntry, PrivacyConfig,
                                   # MaskMode ('mask' | 'redact' | 'route')
      detector.ts                  # PIIDetector class:
                                   #   - detectStructured(text): regex-based (OpenRedaction)
                                   #   - detectUnstructured(text): NER-based (transformers.js)
                                   #   - detect(text): merge + deduplicate detections
                                   #   - Lazy model loading (NER model downloaded on first call)
                                   #   - Confidence threshold filtering
      substitutor.ts               # PIISubstitutor class:
                                   #   - generate(type, seed): type-preserving fake via Faker
                                   #   - Seeded Faker instance for session consistency
      vault.ts                     # PrivacyVault class:
                                   #   - getOrCreate(realValue, type): idempotent mapping
                                   #   - restore(maskedText): longest-match-first restoration
                                   #   - clear(): session cleanup
                                   #   - Bidirectional Map<real, fake> + Map<fake, real>
      gateway.ts                   # PrivacyGateway class:
                                   #   - mask(text): detect -> substitute -> return masked text
                                   #   - unmask(text): vault.restore()
                                   #   - wrapQuery(queryFn): higher-order function wrapping query()
                                   #   - Whitelist filtering
                                   #   - Code block conservative mode
      stream-handler.ts            # StreamRestorer class:
                                   #   - pipe(stream): buffer chunks, restore at token boundaries
                                   #   - Handles partial fake-data tokens across chunk boundaries
      hooks.ts                     # Claude Agent SDK hook integration:
                                   #   - privacyMaskHook (PreToolUse)
                                   #   - privacyUnmaskHook (PostToolUse)
      index.ts                     # Public API exports
    __tests__/
      detector.test.ts             # PII detection accuracy per type, confidence thresholds
      vault.test.ts                # Bidirectional mapping, session consistency, longest match
      gateway.test.ts              # End-to-end mask/unmask, whitelist, code block handling
      stream-handler.test.ts       # Streaming restoration, chunk boundary handling
```

### Key Implementation Details
- **Longest match restoration**: Sort vault entries by key length descending before replacement
- **Code block detection**: Regex to identify fenced code blocks; apply conservative detection within them
- **Lazy NER loading**: First call to `detectUnstructured()` triggers model download; subsequent calls use cached model
- **Whitelist**: Pre-filter detections before substitution

### Testing Strategy
- Unit tests per PII type (email, phone, credit card, API key, names, etc.)
- Vault consistency: same input always maps to same output within session
- Round-trip tests: mask -> unmask produces original text
- Streaming tests: chunked input with fake data spanning chunk boundaries
- Whitelist tests: whitelisted terms are never masked
- Code block tests: PII in code blocks uses conservative strategy

### Exit Criteria
- [ ] Detects all configured PII types with >85% precision
- [ ] Round-trip mask/unmask preserves original text exactly
- [ ] Streaming restoration handles chunk boundaries correctly
- [ ] Whitelist exclusion works
- [ ] NER model downloads on demand, cached for subsequent use

---

## Phase 3: Memory System

**Goal**: Full memory stack — Core blocks, Recall (dual-write + hybrid search), ACL, provenance.

**Estimated Complexity**: High (~2 weeks)

**Depends on**: Phase 1 (types, config)

### Technical Decisions
- **SQLite**: `better-sqlite3` (synchronous, fast, zero external deps)
- **Vector search**: `sqlite-vec` extension (native SQLite vector similarity)
- **Full-text search**: SQLite FTS5 (built-in, BM25 scoring)
- **Embedding**: `@xenova/transformers` (transformers.js, all-MiniLM-L6-v2 ONNX, ~100MB)
- **Hybrid retrieval**: Reciprocal Rank Fusion (k=60) merging vector + BM25 results

### File Breakdown

```
packages/
  memory/
    package.json                   # @opc/memory
    tsconfig.json
    tsup.config.ts
    src/
      types.ts                     # MemoryBlock, RecallEntry, SemanticFact, SearchResult,
                                   # MemoryProvenance, Namespace, ACLPermission
      embedding.ts                 # EmbeddingProvider interface + implementations:
                                   #   - LocalEmbedding (transformers.js, lazy model load)
                                   #   - OpenAIEmbedding (API-based)
                                   #   - VoyageEmbedding (API-based)
                                   #   Factory: createEmbeddingProvider(config)
      blocks.ts                    # MemoryBlockStore class:
                                   #   - get/set/update block by agent + label
                                   #   - Optimistic locking via version column
                                   #   - readOnly enforcement
                                   #   - Character limit validation
                                   #   - Shared block support (agentId = null)
      recall.ts                    # RecallMemory class:
                                   #   - write(message): sync SQL + async vector embedding
                                   #   - search(query, agentId): hybrid vector + BM25
                                   #   - Role-based embedding format
                                   #   - Content hash dedup
      hybrid-search.ts             # HybridSearch class:
                                   #   - vectorSearch(query, k): sqlite-vec cosine similarity
                                   #   - bm25Search(query, k): FTS5
                                   #   - fuse(vectorResults, bm25Results, k): RRF (k=60)
      acl.ts                       # MemoryACL class:
                                   #   - checkPermission(requester, target, operation): bool
                                   #   - Namespace parsing: /org/agent/{id}/{tier}
                                   #   - Rules: self=full, child=read+write, parent=none,
                                   #     sibling=shareWith only
                                   #   - Shared block: referenced=read, !readOnly=write
      provenance.ts                # ProvenanceTracker:
                                   #   - attach(entry, metadata): add provenance to memory entry
                                   #   - query(filters): search by author, taskId, threadId
                                   #   - Content hash generation (MD5)
      adapters/
        adapter.ts                 # StorageAdapter interface (CRUD for blocks, recall, semantic)
        sqlite.ts                  # SQLiteAdapter class:
                                   #   - Schema: blocks, recall_messages, recall_vectors,
                                   #     semantic_facts, semantic_vectors, provenance
                                   #   - sqlite-vec for vector ops
                                   #   - FTS5 virtual table for BM25
                                   #   - Migration runner
      index.ts                     # MemorySystem facade: init, getBlocks, getRecall, etc.
    __tests__/
      blocks.test.ts               # CRUD, optimistic lock conflicts, readOnly, shared blocks
      recall.test.ts               # Write + search, hybrid retrieval accuracy, dedup
      hybrid-search.test.ts        # RRF fusion correctness, ranking quality
      acl.test.ts                  # Full permission matrix (self/child/parent/sibling x read/write)
      provenance.test.ts           # Metadata attachment and querying
      sqlite.test.ts               # Adapter: schema creation, migrations, CRUD operations
```

### Key Implementation Details
- **SQLite schema**: Single database file per OPC instance. Tables: `memory_blocks`, `recall_messages`, `recall_vectors` (sqlite-vec virtual table), `semantic_facts`, `provenance`
- **Async embedding**: After synchronous SQL write, queue embedding job. Use a simple in-process task queue (no external deps).
- **RRF formula**: `score = sum(1 / (k + rank_i))` where k=60, across vector and BM25 result sets
- **ACL enforcement**: Middleware layer that wraps all memory reads/writes; checks namespace permissions before executing

### Testing Strategy
- Unit tests for each memory tier independently
- ACL: exhaustive combinatorial tests (all requester/target/operation combinations)
- Hybrid search: test that combining vector + BM25 outperforms either alone on synthetic queries
- Optimistic locking: concurrent write conflict detection
- Integration: full flow from block edit -> recall write -> search -> ACL filtering

### Exit Criteria
- [ ] Memory blocks: CRUD with optimistic locking works
- [ ] Recall: dual-write + hybrid search returns relevant results
- [ ] ACL: all permission matrix combinations pass
- [ ] Provenance: entries carry correct metadata
- [ ] SQLite adapter handles all operations with zero external dependencies

---

## Phase 4: Orchestration Engine

**Goal**: The delegate tool, multi-layer orchestration, context assembly, event bus.

**Estimated Complexity**: High (~2 weeks)

**Depends on**: Phase 1 (types, registry), Phase 3 (memory)

### Technical Decisions
- **Agent execution**: Claude Agent SDK `query()` function
- **Concurrency**: `Promise.allSettled()` for parallel delegation with configurable max concurrency via semaphore
- **Result compression**: Claude Haiku for summarizing results >2000 tokens
- **Event system**: Node.js EventEmitter-based EventBus (simple, no external deps)

### File Breakdown

```
packages/
  core/
    src/
      orchestrator.ts              # Orchestrator class:
                                   #   - execute(task): entry point, creates root query
                                   #   - delegate(parentSession, targetAgentId, task): validate
                                   #     route, create child session, run query(), compress result
                                   #   - Session tracking: Map<sessionId, SessionState>
                                   #   - Concurrency control: semaphore (maxConcurrentAgents)
                                   #   - Depth tracking: reject if > maxHierarchyDepth
                                   #   - Timeout enforcement per task
      router.ts                    # TaskRouter class:
                                   #   - route(parentAgentId, targetAgentId): validate parent-child
                                   #   - Uses registry adjacency graph
      context-builder.ts           # ContextBuilder class:
                                   #   - build(agentId, task): assemble full context
                                   #   - write(): system prompt + core blocks + org structure
                                   #   - select(): recall search + semantic search + parent task
                                   #   - compress(): token budget enforcement, summarize if needed
                                   #   - isolate(): independent window, sibling invisible
      event-bus.ts                 # EventBus class:
                                   #   - emit(event): publish to subscribers
                                   #   - on(eventType, handler): subscribe
                                   #   - Events: SubagentStart, SubagentStop, TaskDelegated,
                                   #     TaskCompleted, TaskFailed, MemoryUpdated

  tools/
    package.json                   # @opc/tools
    tsconfig.json
    tsup.config.ts
    src/
      delegate.ts                  # delegate MCP tool definition:
                                   #   - Input: targetAgentId, taskDescription
                                   #   - Validates route via router
                                   #   - Calls orchestrator.delegate()
                                   #   - Returns compressed result string
      delegate-parallel.ts         # delegateParallel MCP tool:
                                   #   - Input: Array<{agentId, task}>
                                   #   - Runs Promise.allSettled on multiple delegate calls
                                   #   - Returns aggregated results
      memory-replace.ts            # memory_replace tool: block label + old + new text
      memory-insert.ts             # memory_insert tool: block label + text to append
      memory-read.ts               # memory_read tool: search query -> recall + semantic results
      escalate.ts                  # escalate tool: type + reason + partialResult -> parent
      human-approval.ts            # human_approval tool: question + options -> wait for human
      index.ts
    __tests__/
      delegate.test.ts             # Route validation, result compression, timeout
      orchestrator.test.ts         # Multi-layer chains, concurrency limits, depth limits
      context-builder.test.ts      # W/S/C/I pipeline, token budgets
```

### Key Implementation Details
- **Delegate as tool**: Defined as an MCP tool schema. When called, the framework intercepts it and routes through Orchestrator rather than executing as a regular tool
- **Session management**: Each `query()` gets a unique sessionId. Orchestrator tracks active sessions for cleanup on timeout
- **Result compression**: If child result > `resultCompressionThreshold` tokens, call Haiku with "Summarize this result: ..." before returning to parent
- **Context budget**: ContextBuilder allocates token budget — e.g., 60% for system+blocks, 20% for recall, 10% for semantic, 10% buffer

### Testing Strategy
- Unit: delegate route validation (valid child, invalid sibling, invalid grandchild)
- Unit: ContextBuilder W/S/C/I pipeline produces expected context shape
- Integration: 3-layer delegation chain (mock agents) — CEO -> VP -> Dev
- Concurrency: verify maxConcurrentAgents limit is enforced
- Timeout: verify tasks are killed after taskTimeout
- Result compression: verify large results are summarized

### Exit Criteria
- [ ] Single delegate call works end-to-end
- [ ] 3-layer delegation chain completes successfully
- [ ] Parallel delegation works with concurrency limits
- [ ] Context builder produces well-structured context within token budgets
- [ ] EventBus emits events for all lifecycle stages

---

## Phase 5: Semantic Memory + Consolidation

**Goal**: Fact extraction pipeline, dedup, graph memory, sleep-time consolidation.

**Estimated Complexity**: High (~1.5 weeks)

**Depends on**: Phase 3 (memory system, embedding, SQLite)

### Technical Decisions
- **Fact extraction**: LLM-based (Claude Haiku for cost efficiency) with structured output
- **Graph storage**: SQLite relational tables by default (entities + relationships tables); Neo4j adapter optional
- **Bi-temporal tracking**: `valid_time` (when fact was true) + `recorded_time` (when stored)
- **Consolidation schedule**: Triggered after N interactions via counter, runs as async task

### File Breakdown

```
packages/
  memory/
    src/
      semantic.ts                  # SemanticMemory class:
                                   #   - extract(conversation): LLM fact extraction
                                   #   - store(facts): embed + conflict detect + write
                                   #   - search(query): vector similarity search
                                   #   - 5-stage pipeline: extract -> embed -> conflict ->
                                   #     decide (ADD/UPDATE/DELETE/NOOP) -> write
                                   #   - Dedup: content hash -> semantic threshold (0.7) ->
                                   #     LLM judgment
      graph.ts                     # GraphMemory class:
                                   #   - addEntity(name, type, embedding): create node
                                   #   - addRelation(source, target, type): create edge
                                   #   - search(query): BM25 rerank on graph triples
                                   #   - Default: SQLite tables (entities, relationships)
                                   #   - Optional: Neo4j adapter
      consolidator.ts              # SleepTimeConsolidator class:
                                   #   - shouldRun(agentId): check interaction counter
                                   #   - run(agentId): process unprocessed recall entries
                                   #     1. Fetch unprocessed recall messages
                                   #     2. Run fact extraction pipeline
                                   #     3. Reflect on core memory blocks, suggest updates
                                   #     4. Mark entries as processed
                                   #   - Uses low-cost model (Haiku)
    __tests__/
      semantic.test.ts             # Fact extraction quality, conflict resolution, dedup
      graph.test.ts                # Entity/relation CRUD, triple search
      consolidator.test.ts         # Trigger logic, full consolidation flow
```

### Testing Strategy
- Fact extraction: test with synthetic conversations, verify extracted facts are accurate
- Conflict resolution: test ADD/UPDATE/DELETE decisions against known scenarios
- Dedup: verify 4-layer dedup catches duplicates at each level
- Graph: entity/relation CRUD, search returns relevant triples
- Consolidation: end-to-end — insert recall entries, trigger consolidation, verify semantic memory populated

### Exit Criteria
- [ ] Fact extraction produces reasonable atomic facts from conversations
- [ ] Conflict resolution correctly handles ADD/UPDATE/DELETE
- [ ] Dedup prevents duplicate facts across all 4 layers
- [ ] Graph memory stores and searches entity-relationship triples
- [ ] Sleep-time consolidator processes backlog and updates core memory

---

## Phase 6: Communication Integration

**Goal**: Chat SDK gateway, streaming bridge, JSX cards, webhook server.

**Estimated Complexity**: Medium (~1 week)

**Depends on**: Phase 4 (orchestrator, event bus)

### Technical Decisions
- **Chat SDK**: Vercel Chat SDK (`@vercel/chat-sdk` or equivalent)
- **Webhook server**: Next.js API routes (consistent with dashboard)
- **JSX cards**: React components rendered to platform-specific formats
- **Streaming**: AsyncGenerator-to-AsyncIterable bridge

### File Breakdown

```
packages/
  gateway/
    package.json                   # @opc/gateway
    tsconfig.json
    src/
      chat-gateway.ts              # ChatGateway class:
                                   #   - onNewMention(event): route to root agent
                                   #   - onSubscribedMessage(event): restore session
                                   #   - onAction(event): handle button clicks
                                   #   - onSlashCommand(event): /status, /agents, etc.
                                   #   - Session restore from thread context
      stream-bridge.ts             # StreamBridge:
                                   #   - bridge(queryGenerator): AsyncGenerator -> AsyncIterable
                                   #   - Handles backpressure
                                   #   - Error propagation
      cards/
        status-card.tsx            # Task status card with progress
        approval-card.tsx          # Approval request with Accept/Reject buttons
        agent-tree-card.tsx        # Compact agent hierarchy view
      index.ts
    __tests__/
      chat-gateway.test.ts         # Event routing, session management
      stream-bridge.test.ts        # Streaming correctness, backpressure

apps/
  server/
    package.json
    src/app/api/
      slack/route.ts               # Slack webhook handler -> ChatGateway
      discord/route.ts             # Discord webhook handler -> ChatGateway
```

### Testing Strategy
- Unit: ChatGateway routes events to correct handlers
- Unit: StreamBridge converts AsyncGenerator to AsyncIterable correctly
- Integration: simulate incoming webhook -> gateway -> orchestrator -> stream response

### Exit Criteria
- [ ] Gateway handles all event types (mention, message, action, slash command)
- [ ] Stream bridge delivers real-time responses
- [ ] JSX cards render correctly
- [ ] Webhook routes process platform-specific payloads

---

## Phase 7: Web UI Dashboard

**Goal**: Full dashboard with agent tree, task flow, memory viewer, config editor, logs.

**Estimated Complexity**: High (~2 weeks)

**Depends on**: Phase 4 (event bus), Phase 3 (memory), Phase 1 (config)

### Technical Decisions
- **Framework**: Next.js 15 (App Router)
- **UI components**: shadcn/ui (accessible, composable) + BaseUI (data-heavy components)
- **Graph visualization**: React Flow (agent tree + task flow)
- **Real-time**: WebSocket via EventBus subscription
- **State management**: React Query / SWR for server state
- **Code editor**: Monaco Editor for YAML config editing

### File Breakdown

```
apps/
  dashboard/
    package.json
    next.config.ts
    tailwind.config.ts
    src/
      app/
        layout.tsx                 # Root layout with sidebar navigation
        page.tsx                   # Dashboard overview: active tasks, agent status, timeline
        agents/
          page.tsx                 # Agent Tree: React Flow interactive hierarchy
        tasks/
          page.tsx                 # Task Flow: real-time delegation chain visualization
        memory/
          page.tsx                 # Memory Viewer: tabs for Core/Recall/Semantic/Graph
        config/
          page.tsx                 # Config Editor: Monaco YAML editor + schema validation
        logs/
          page.tsx                 # Logs: filterable log aggregation
        privacy/
          page.tsx                 # Privacy Dashboard: masking stats, vault status
      components/
        sidebar.tsx                # Navigation sidebar
        agent-node.tsx             # React Flow custom node for agents
        task-node.tsx              # React Flow custom node for tasks
        memory-block-editor.tsx    # Editable Core Memory block component
        recall-search.tsx          # Recall memory search interface
        semantic-list.tsx          # Semantic fact list with filters
        log-viewer.tsx             # Log entries with filtering
      hooks/
        use-websocket.ts           # WebSocket connection to EventBus
        use-agents.ts              # Agent data fetching
        use-tasks.ts               # Task data fetching
        use-memory.ts              # Memory data fetching
      lib/
        api.ts                     # API client for backend
```

### Testing Strategy
- Component tests: render each page with mock data
- Integration: WebSocket connection receives real-time events
- E2E (optional): Playwright tests for critical flows

### Exit Criteria
- [ ] All 7 pages render with correct data
- [ ] Agent tree is interactive (click to view details)
- [ ] Task flow updates in real-time via WebSocket
- [ ] Memory viewer allows browsing all three tiers
- [ ] Config editor validates YAML against schema

---

## Phase 8: Production Hardening

**Goal**: Production-grade storage adapters, containerization, docs, CI/CD.

**Estimated Complexity**: Medium (~1 week)

**Depends on**: All previous phases

### Technical Decisions
- **Postgres**: `pg` + `pgvector` extension for vector search, native FTS for BM25
- **Neo4j**: Official `neo4j-driver` for full graph capabilities
- **Container**: Docker Compose with profiles (dev: SQLite, prod: Postgres+Neo4j)
- **CI/CD**: GitHub Actions (lint, test, build, publish)

### File Breakdown

```
packages/
  memory/
    src/
      adapters/
        postgres.ts                # PostgresAdapter: pgvector + FTS
        neo4j.ts                   # Neo4jAdapter: Cypher queries for graph memory

docker-compose.yml                 # Services: opc-server, dashboard, postgres, neo4j (profiles)
Dockerfile                         # Multi-stage build for server + dashboard

.github/
  workflows/
    ci.yml                         # Lint + test + build on PR
    release.yml                    # Publish to npm on tag

docs/
  getting-started.md               # Quick start guide
  configuration.md                 # Config reference
  architecture.md                  # Architecture overview
  deployment.md                    # Deployment guide
```

### Testing Strategy
- Postgres adapter: run against Docker Postgres with pgvector
- Neo4j adapter: run against Docker Neo4j
- Docker Compose: smoke test full stack startup
- CI: all tests pass in CI environment

### Exit Criteria
- [ ] Postgres adapter passes all memory tests
- [ ] Neo4j adapter handles graph operations
- [ ] Docker Compose starts all services
- [ ] CI/CD pipeline runs successfully
- [ ] Documentation covers setup, config, and deployment

---

## Dependency Graph

```
Phase 1 (Foundation) ─────┬──> Phase 2 (Privacy)
                          ├──> Phase 3 (Memory) ──────┬──> Phase 5 (Semantic + Consolidation)
                          │                           │
                          └──> Phase 4 (Orchestration) ┤
                               (needs Phase 1 + 3)     ├──> Phase 6 (Communication)
                                                       │    (needs Phase 4)
                                                       │
                                                       ├──> Phase 7 (Dashboard)
                                                       │    (needs Phase 3 + 4)
                                                       │
                                                       └──> Phase 8 (Production)
                                                            (needs all)
```

**Parallelizable**: Phase 2 (Privacy) and Phase 3 (Memory) can run in parallel after Phase 1 completes.

---

## Summary

| Phase | Description | Complexity | Est. Duration | Key Deliverables |
|-------|------------|-----------|---------------|-----------------|
| 1 | Foundation Skeleton | Low-Med | ~1 week | Monorepo, types, config, registry, CLI |
| 2 | Privacy Gateway | Med-High | ~1.5 weeks | PII detection, vault, mask/unmask, streaming |
| 3 | Memory System | High | ~2 weeks | Blocks, recall, hybrid search, ACL, provenance |
| 4 | Orchestration Engine | High | ~2 weeks | Delegate, orchestrator, context builder, events |
| 5 | Semantic + Consolidation | High | ~1.5 weeks | Fact extraction, dedup, graph, sleep-time |
| 6 | Communication | Medium | ~1 week | Chat gateway, stream bridge, JSX cards |
| 7 | Web UI Dashboard | High | ~2 weeks | 7 dashboard pages, real-time updates |
| 8 | Production Hardening | Medium | ~1 week | Postgres, Neo4j, Docker, docs, CI/CD |
| | **Total** | | **~12 weeks** | |
