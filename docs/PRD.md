# OPC - Hierarchical Multi-Agent Framework

## Context

Build an open-source framework that allows users to dispatch a group of AI Agents — organized by company hierarchy — through chat platforms (Slack/Discord/Teams, etc.). Humans communicate with the top-level Agent; top-level Agents only make decisions and dispatch; specific tasks are delegated down through the layers for execution. Core focus areas: context engineering, hierarchical memory system, cross-agent memory access control.

Tech stack: **Claude Agent SDK** (agent execution engine) + **Vercel Chat SDK** (multi-platform communication layer), pure TypeScript.

### Design Principles
- **Local-first**: Zero cloud dependencies by default (SQLite + local embedding + local NER), with optional upgrade to production-grade backends
- **Privacy built-in**: All LLM API calls pass through a local privacy gateway; PII is automatically masked/restored; real data never leaves the local environment
- **Lightweight > Comprehensive**: Avoid bloat like OpenClaw; focus only on core orchestration + memory + communication

---

## 1. Core Architecture: N-Layer Agent Hierarchy

Claude Agent SDK native limitation: subagents cannot create further subagents (only supports 2 layers).

**Solution: Recursive `query()` calls via a `delegate` MCP tool**

Each layer is not a nested subagent but an independent `query()` call. `delegate` appears to the parent agent as just a tool that returns results, but internally the framework launches a complete agent session.

```
Human -> [Chat SDK] -> CEO Agent (query #1)
                          |  calls delegate("vp-eng", task)
                       VP Eng (query #2)      <- Independent query() call
                          |  calls delegate("lead-backend", task)
                       Lead Backend (query #3) <- Another independent query()
                          |  Executes directly
```

Each `query()` has its own independent session, context window, and tool set. `delegateParallel` supports dispatching multiple child agents simultaneously.

---

## 2. Agent Hierarchy Model - YAML Configuration

```yaml
# opc.config.yaml
framework:
  name: "My AI Org"
  defaultModel: "claude-sonnet-4-20250514"

  memory:
    # Default lightweight mode (SQLite + built-in vectors)
    storage: "sqlite"              # sqlite | postgres
    embedding:
      provider: "openai"           # openai | voyage | local
      model: "text-embedding-3-small"
      dimensions: 512              # Reducing to 512 maintains ~98% accuracy
    graph:
      enabled: false               # Requires Neo4j when true
    consolidation:
      enabled: true
      frequency: 5                 # Trigger sleep-time consolidation every 5 interactions
      model: "claude-haiku-4-5-20251001"

  execution:
    maxConcurrentAgents: 10
    maxHierarchyDepth: 6
    taskTimeout: "5m"
    resultCompressionThreshold: 2000  # tokens

channels:
  slack:
    adapter: "@chat-adapter/slack"
  discord:
    adapter: "@chat-adapter/discord"

agents:
  ceo:
    role: "CEO"
    level: 0
    prompt: "file:agents/ceo.md"
    model: "claude-opus-4-6"
    tools: [delegate, delegateParallel, memory_read, memory_write]
    children: [vp-eng, vp-ops]
    memory:
      blocks:                        # Core Memory blocks (always in context)
        - label: "persona"
          description: "Your identity and decision-making style"
          value: ""
          limit: 2000
        - label: "org_context"
          description: "Current organizational priorities and ongoing projects"
          value: ""
          limit: 3000
      recall: true                   # Conversation history semantic search
      archival: true                 # Long-term fact extraction
      shareWith: []

  vp-eng:
    role: "VP Engineering"
    level: 1
    parent: ceo
    prompt: "file:agents/vp-eng.md"
    tools: [delegate, delegateParallel, memory_read, memory_write, Grep, Glob]
    children: [lead-backend, lead-frontend]
    memory:
      blocks:
        - label: "persona"
          value: ""
          limit: 2000
        - label: "team_status"
          description: "Current team workload and project status"
          value: ""
          limit: 3000
      recall: true
      archival: true
      sharedBlocks: [team_policies]  # Reference to shared block
      shareWith: [vp-ops]

  dev-1:
    role: "Senior Dev"
    level: 3
    parent: lead-backend
    tools: [memory_read, memory_write, Read, Write, Edit, Bash, Grep, Glob]
    children: []
    memory:
      blocks:
        - label: "persona"
          value: ""
          limit: 1000
      recall: true
      archival: false                # Leaf nodes don't need long-term graph

# Shared memory block definitions (cross-agent)
sharedBlocks:
  team_policies:
    description: "Engineering team policies and coding standards"
    value: ""
    limit: 5000
    readOnly: false                  # false = agents can edit
```

Startup validation: no cycles, no orphans, shareWith at same level, leaf nodes have no delegate, sharedBlocks must exist.

---

## 3. Communication Layer - Vercel Chat SDK Integration

Chat SDK serves as the sole human-machine interaction entry point:

- `onNewMention` -> New conversation, routes to root agent
- `onSubscribedMessage` -> Follow-up messages, restores existing session
- `onAction` -> Button interactions (approvals, cancellations, etc.)
- `onSlashCommand` -> `/status`, `/agents`, etc.

**Message Flow**:
```
Human sends message -> Chat SDK -> Gateway -> Orchestrator -> query(CEO)
CEO delegate -> query(VP) -> ... -> Leaf agent executes
Results compressed and propagated back up -> CEO formats -> Chat SDK stream -> Human sees reply
```

Streaming output: `query()` AsyncGenerator -> bridge -> AsyncIterable<string> -> `thread.post()` native streaming consumption.

---

## 4. Privacy Gateway - Automatic PII Masking/Restoration

A local transparent proxy between Agents and the LLM API. Sensitive information is automatically replaced with fake data before being sent out, then restored on return. Agents are unaware; real data never leaves the local environment.

### Workflow

```
Agent query()
    |
Privacy Gateway (local in-process middleware)
    |- 1. PII Detection: NER model (names/orgs/addresses) + 570+ regex (email/phone/API Key/credit card...)
    |- 2. Generate Mapping: real value -> same-type fake data (Faker.js, consistent within session)
    |- 3. Replace: PII in prompt -> fake data
    |- 4. Send: masked prompt -> LLM API
    |- 5. Receive: LLM response (containing fake data references)
    |- 6. Restore: fake data -> real values (longest match first to avoid partial replacement)
    +- 7. Return: restored response -> Agent
```

### Tech Stack (pure TypeScript, zero Python dependencies)

| Layer | Library | Purpose |
|-------|---------|---------|
| **PII Detection - Structured** | OpenRedaction | 570+ regex patterns: email, phone, credit card (Luhn), API Key, JWT, SSH Key, IP |
| **PII Detection - Unstructured** | PII-PALADIN | BERT NER via transformers.js (ONNX): person names, organization names, addresses, pure local inference. **Downloaded on demand**: automatically downloads ~400MB model on first use when privacy is enabled |
| **Fake Data Generation** | @faker-js/faker | Same-type replacement: name->name, email->email. `faker.seed()` ensures consistency within session |
| **Mapping Storage** | In-memory Map | One Vault per session/thread: `Map<realValue, fakeValue>`, bidirectional lookup |

### Vault (Mapping Table) Design

```typescript
interface PrivacyVault {
  sessionId: string;
  mappings: Map<string, string>;     // real -> fake
  reverseMappings: Map<string, string>; // fake -> real
  entityTypes: Map<string, PIIType>; // real -> type (PERSON, EMAIL, etc.)
}
```

- **Session-level scope**: Within the same session, "Zhang San" always maps to the same fake name "Alex Chen"
- **Type preservation**: email->email, phone->phone, address->address
- **Longest match first**: During restoration, replace "Zhang Sanfeng" before "Zhang San" to avoid incorrect partial replacement
- **Vault is not persisted**: Automatically cleared after session ends, leaving no traces

### Configuration

```yaml
# opc.config.yaml
framework:
  privacy:
    enabled: true                  # Global toggle
    detectTypes:                   # PII types to detect
      - PERSON_NAME
      - EMAIL
      - PHONE
      - CREDIT_CARD
      - API_KEY
      - ADDRESS
      - SSN
      - IP_ADDRESS
    whitelist:                     # Terms exempt from masking (public info like company names)
      - "Anthropic"
      - "Claude"
    confidenceThreshold: 0.85     # NER confidence threshold
    mode: "mask"                  # mask (replace with fake data) | redact (replace with [REDACTED]) | route (route sensitive requests to local model)
```

### Integration Method

Transparently injected as Claude Agent SDK hooks or MCP middleware, without modifying agent code:

```typescript
// Intercept before and after query() calls
hooks: {
  PreToolUse: [{ matcher: ".*", hooks: [privacyMaskHook] }],
  PostToolUse: [{ matcher: ".*", hooks: [privacyUnmaskHook] }],
}
```

### Known Limitations and Mitigations

| Issue | Mitigation |
|-------|-----------|
| NER name detection is imperfect (~90% F1) | Combine regex + NER + whitelist; prefer over-masking |
| "Apple's CEO" — should Apple be masked? | User-configured whitelist to exclude public entities |
| LLM may make incorrect inferences based on fake data | Use Faker to generate realistic same-type data, maintaining semantic consistency |
| PII replacement in code blocks may break syntax | Use conservative strategy for code block regions (only replace obvious keys/connection strings) |
| Restoration of streaming responses | Buffer to complete token boundaries before performing string replacement |

---

## 5. Memory System - Three-Layer Architecture

### Research Conclusions

| Framework | Core Innovation | Deployment | TS SDK | Multi-Agent | Lightest Deps |
|-----------|----------------|-----------|--------|-------------|---------------|
| **Letta** | Agent self-editing memory blocks + sleep-time | Server (REST) | Yes | Yes, shared blocks | SQLite + Ollama |
| **Mem0** | LLM conflict resolution writes + 4-level scope | Library (embedded) | Yes | Yes, 4-level isolation | Qdrant + Ollama |
| **Zep/Graphiti** | Bi-temporal fact tracking | Library | No, Python only | No | Neo4j + LLM |
| **EverMemOS** | Brain-inspired memory consolidation SOTA | Server | No | No | MongoDB+ES+Milvus+Redis |
| **OpenViking** | File system paradigm L0/L1/L2 | Library+Server | No, Python only | No | Local files + LLM |

**OPC Memory System Adopts**:
- Core Memory -> **Letta's Memory Blocks** (agent self-editing, shared blocks, optimistic locking)
- Recall Memory -> **Letta's dual-write mode** (SQL for ordering + Vector for search) + **hybrid retrieval RRF**
- Semantic Memory -> **Mem0's fact extraction pipeline** (LLM conflict resolution ADD/UPDATE/DELETE)
- Time awareness -> Inspired by **Zep's bi-temporal axis** (recording fact's valid_time and recorded_time)
- ACL -> **AWS Bedrock AgentCore's namespace hierarchy**
- Consolidation -> **Letta's sleep-time compute**

### Local-First Solution Confirmation

| Component | Default Local Solution | External Dependencies |
|-----------|----------------------|----------------------|
| Core Memory (blocks) | SQLite (better-sqlite3) | None |
| Recall Memory (vectors) | SQLite + sqlite-vec extension | None (JS fallback available) |
| Recall Memory (keywords) | SQLite FTS5 | None (built into SQLite) |
| Semantic Memory (facts) | SQLite + sqlite-vec | None |
| Graph Memory | SQLite relational tables (simplified) | None (Neo4j optional upgrade) |
| Embedding | **transformers.js + all-MiniLM-L6-v2 ONNX (default)** | No cloud API (~100MB, auto-download on first use) |
| Fact Extraction LLM | Local Ollama or cloud API | Ollama optional |
| NER Model | transformers.js + bert-base-NER ONNX | No cloud API (~400MB, download on demand) |

**Fully offline capable**: SQLite suite + transformers.js local embedding + Ollama local LLM. No cloud APIs or external services required.

### Architecture Overview

```
+---------------------------------------------------------+
|  Core Memory (Memory Blocks)                            |
|  Always in agent context; agent self-edits via tool calls|
|  Format: labeled blocks with character limits            |
|  Storage: SQL (blocks table)                             |
|  Sharing: multiple agents reference the same block ID    |
+---------------------------------------------------------+
                         |
+---------------------------------------------------------+
|  Recall Memory (Episodic / Conversation History)        |
|  Semantic search over conversation history               |
|  Storage: SQL (metadata/ordering) + Vector DB (search)   |
|  Write: sync SQL + async embedding                       |
|  Retrieval: hybrid - 70% vector similarity + 30% BM25   |
|  Dedup: content hash + semantic threshold (0.7)          |
+---------------------------------------------------------+
                         |
+---------------------------------------------------------+
|  Semantic Memory (Long-term Facts & Knowledge)          |
|  Structured facts extracted from conversations           |
|  Pipeline: fact extraction -> conflict detection ->      |
|            ADD/UPDATE/DELETE                              |
|  Storage: Vector DB + optional Graph DB (Neo4j)          |
|  Dedup: 4 layers - hash / semantic / LLM judgment / scope|
|  Consolidation: Sleep-time consolidation (async bg)      |
+---------------------------------------------------------+
```

### 4.1 Core Memory - Memory Blocks (Inspired by Letta)

Each agent owns multiple labeled memory blocks, **always prepended to system prompt**:

```typescript
interface MemoryBlock {
  id: string;
  label: string;           // "persona", "org_context", "team_status"
  description: string;     // Explains purpose, helps agent understand when to use it
  value: string;           // Actual content
  limit: number;           // Character limit
  readOnly: boolean;       // true = agent cannot modify (suitable for subordinate agents)
  version: number;         // Optimistic lock, prevents concurrent conflicts
  agentId: string;         // Owning agent (null = shared block)
}
```

Agents self-edit via tool calls:
- `memory_replace(block_label, old_text, new_text)` — Precise replacement
- `memory_insert(block_label, new_text)` — Append content
- Each edit validates: value.length <= limit, block exists, not readOnly

**Shared blocks**: Multiple agents reference the same block ID. One agent's update is immediately visible to all references (optimistic lock + version column).

### 4.2 Recall Memory - Conversation History (Inspired by Letta's Dual-Write Mode)

**Dual-write strategy**:
- Synchronous write to SQL (guarantees message ordering and metadata integrity)
- Asynchronous write to Vector DB (embedding doesn't block agent response)

**Message embedding strategy** (differentiated by role):

| Role | Embedding Format |
|------|-----------------|
| user | `{"content": "user message text"}` |
| assistant + tool_call | `{"thinking": "CoT", "content": "reply"}` |
| tool result | `{"tool_call": "...", "tool_result": "..."}` |
| system | Skipped (not embedded) |

**Retrieval**: Hybrid strategy
```
Query -> [Vector Search top-K] + [BM25/FTS5 top-K]
      -> Reciprocal Rank Fusion (k=60)
      -> Return top-N results
```

### 4.3 Semantic Memory - Fact Extraction (Inspired by Mem0 Pipeline)

**5-stage pipeline**:
1. **Fact Extraction**: LLM extracts atomic facts from conversations (preferences, decisions, entity info)
2. **Embedding**: Vectorize each fact
3. **Conflict Detection**: Top-10 similarity search against existing semantic memory
4. **Action Decision**: LLM determines each fact should `ADD` (new) / `UPDATE` (update existing) / `DELETE` (remove outdated) / `NOOP` (ignore)
5. **Parallel Write**: Simultaneously write to Vector DB + optional Graph DB

**Graph Memory** (optional, when Neo4j is enabled):
- Entity nodes: name, type, embedding, agent_id, timestamps
- Relationship edges: typed relationships (LIVES_IN, PREFERS, DEPENDS_ON...)
- Search: BM25 reranking on graph triples

### 4.4 Hierarchical Access Control (Namespace-based ACL)

Inspired by AWS Bedrock AgentCore's namespace hierarchy pattern:

```
/org/
  /agent/{agentId}/
    /core/           -> agent's own memory blocks
    /recall/         -> agent's conversation history
    /semantic/       -> agent's long-term facts
  /shared/
    /{blockLabel}/   -> shared memory blocks
```

**Permission Matrix**:

| Operation | Own Namespace | Child Namespace | Parent Namespace | Sibling Namespace |
|-----------|--------------|-----------------|------------------|-------------------|
| Read Core blocks | Yes | Yes (superior reads subordinate) | No | Configurable via shareWith |
| Write Core blocks | Yes | Yes (issue directives) | No | No |
| Search Recall | Yes | Yes | No | Configurable via shareWith |
| Search Semantic | Yes | Yes | No | Configurable via shareWith |
| Read Shared blocks | Yes (if referenced) | - | - | - |
| Write Shared blocks | Yes (if !readOnly) | - | - | - |

**Implementation**: Each memory entry carries a namespace prefix. memory_read/write tools validate namespace permissions before execution.

### 4.5 Provenance Tracking

Each memory entry carries immutable metadata:

```typescript
interface MemoryProvenance {
  author: string;           // Writing agent ID
  taskId: string;           // Associated task
  threadId: string;         // Source chat thread
  timestamp: number;
  parentTaskId?: string;    // Task chain tracing
  source: 'agent' | 'consolidation' | 'human' | 'system';
  contentHash: string;      // MD5, used for deduplication
}
```

### 4.6 Sleep-time Consolidation (Background Processing)

Inspired by Letta's sleep-time compute pattern:

- **Trigger timing**: Every N agent interactions (configurable, default 5)
- **Executor**: Independent consolidation agent (uses low-cost model haiku)
- **Process**:
  1. Read recent unprocessed messages from agent's Recall memory
  2. Run Mem0-style fact extraction pipeline -> write to Semantic Memory
  3. Reflect on current Core Memory blocks -> generate update suggestions -> auto-apply
  4. Mark processed Recall entries

---

## 5. Context Engineering - Write/Select/Compress/Isolate

Each time `delegate` triggers a new `query()`, context is assembled via pipeline:

1. **Write** (Static injection):
   - System prompt (from agents/{id}/prompt.md)
   - Core Memory blocks (all prepended, always visible)
   - Org structure: "You are {role}, your superior is {parent.role}, your subordinates are {children}"

2. **Select** (Dynamic retrieval):
   - Recall Memory: hybrid search using task description, take top-5
   - Semantic Memory: relevant fact retrieval
   - Parent task instructions: specific task description passed via delegate

3. **Compress** (Window fitting):
   - Core blocks: retained in full (constrained by limits, total is manageable)
   - Recall results: summarized if exceeding token budget
   - Only pass direct parent instructions, not the full ancestor chain

4. **Isolate** (Security boundaries):
   - Each query() has an independent context window
   - Sibling agents are invisible to each other (unless shareWith)
   - Child agent results exceeding 2000 tokens are summarized with a fast model before returning to parent

---

## 6. Message Routing and Task Lifecycle

```
CREATED -> ASSIGNED(root) -> DELEGATED(xN) -> EXECUTING -> RESULT_PROPAGATING -> COMPLETED/FAILED
```

- Parent can only delegate to direct children
- Escalation structured return: `{ type: error|timeout|uncertainty|approval_needed, reason, partialResult }`
- Human approval: agent -> JSX Card button -> human clicks -> resume agent session

---

## 7. Web UI Dashboard

Tech stack: **Next.js 15 + shadcn/ui + BaseUI + React Flow**

### Pages

| Page | Functionality |
|------|-------------|
| **Dashboard** | Active task count, agent status indicators, recent activity timeline |
| **Agent Tree** | React Flow interactive hierarchy graph, click to view details/real-time status |
| **Task Flow** | Real-time task delegation chain visualization, showing current execution position and timing |
| **Memory Viewer** | Browse by agent: Core block editor, Recall search, Semantic fact list, Graph visualization |
| **Config Editor** | Online edit `opc.config.yaml`, schema validation + hierarchy preview |
| **Logs** | Aggregated logs from all agent executions, filterable by task ID / agent / time |

### Real-time Data Flow

```
Orchestrator hooks (SubagentStart/Stop, PostToolUse)
    -> EventBus
    -> WebSocket / SSE
    -> Dashboard real-time updates
```

---

## 8. Project Structure (Monorepo)

```
opc/
  opc.config.yaml                  # User's agent hierarchy configuration
  agents/                          # Agent prompt files
    ceo/prompt.md
    vp-eng/prompt.md

  packages/
    core/                          # @opc/core
      src/
        orchestrator.ts            # Manages query() call chains and sessions
        registry.ts                # AgentRegistry, parses/validates YAML topology
        router.ts                  # TaskRouter, delegate routing logic
        context-builder.ts         # W/S/C/I context assembly pipeline
        event-bus.ts               # Event bus (-> dashboard WebSocket)
        config.ts                  # YAML config parsing and Zod validation
        types.ts                   # Global type definitions

    memory/                        # @opc/memory
      src/
        blocks.ts                  # Core Memory Blocks (Letta pattern)
        recall.ts                  # Recall Memory (dual-write SQL+Vector)
        semantic.ts                # Semantic Memory (Mem0 fact extraction pipeline)
        graph.ts                   # Graph Memory (optional Neo4j)
        acl.ts                     # Namespace-based hierarchical access control
        consolidator.ts            # Sleep-time background consolidation worker
        provenance.ts              # Provenance tracking
        embedding.ts               # Unified embedding interface (local transformers.js + cloud API)
        hybrid-search.ts           # Hybrid retrieval (Vector + BM25 + RRF)
        adapters/
          sqlite.ts                # Default (zero external dependencies)
          postgres.ts              # Production (pgvector + FTS)
          neo4j.ts                 # Graph (optional)

    privacy/                       # @opc/privacy - Privacy Gateway
      src/
        gateway.ts                 # Core middleware: intercept -> mask -> forward -> restore
        detector.ts                # PII detection engine (OpenRedaction regex + PII-PALADIN NER)
        substitutor.ts             # Fake data generation (Faker.js type-preserving substitution)
        vault.ts                   # Session-level mapping table (real <-> fake bidirectional lookup)
        stream-handler.ts          # Streaming response restoration (buffer to token boundary)
        types.ts                   # PIIType, DetectionResult, VaultEntry

    gateway/                       # @opc/gateway
      src/
        chat-gateway.ts            # Chat SDK event handling
        stream-bridge.ts           # query() <-> Chat SDK stream bridge
        cards/                     # JSX Card components
          status-card.tsx
          approval-card.tsx
          agent-tree-card.tsx

    tools/                         # @opc/tools
      src/
        delegate.ts                # N-layer routing core (new query() call)
        delegate-parallel.ts       # Parallel delegation
        memory-replace.ts          # Core block precise replacement
        memory-insert.ts           # Core block append
        memory-read.ts             # Cross-layer memory search (Recall + Semantic)
        escalate.ts                # Escalation
        human-approval.ts          # Request human approval

    cli/                           # @opc/cli
      src/
        commands/
          init.ts                  # Scaffolding generation
          validate.ts              # Config validation
          dev.ts                   # Local development server
          visualize.ts             # Terminal hierarchy tree print

  apps/
    server/                        # Webhook server (Next.js)
      src/app/api/
        slack/route.ts
        discord/route.ts

    dashboard/                     # Web UI (Next.js + shadcn + BaseUI)
      src/app/
        page.tsx                   # Dashboard overview
        agents/page.tsx            # Agent Tree (React Flow)
        tasks/page.tsx             # Task Flow real-time monitoring
        memory/page.tsx            # Memory Viewer
        config/page.tsx            # Config Editor
        logs/page.tsx              # Log aggregation
```

Dependencies: `gateway -> core -> memory`, `tools -> memory`, `core -> privacy`, `dashboard -> core` (read-only), no cycles.

---

## 10. Implementation Steps

### Phase 1: Foundation Skeleton
1. Initialize monorepo (pnpm workspace + turborepo)
2. `@opc/core` type system: AgentDefinition, MemoryBlock, TaskContext, Provenance
3. YAML config parser + Zod schema validator
4. AgentRegistry (parse topology, validate no cycles/no orphans)
5. CLI `opc init` + `opc validate`

### Phase 2: Privacy Gateway
6. PII detection engine: OpenRedaction regex (email/phone/API Key, etc.) + PII-PALADIN NER (person names/orgs/addresses, transformers.js ONNX)
7. Vault mapping table: session-level real<->fake bidirectional Map, type-preserving substitution (Faker.js)
8. Core middleware: intercept request -> mask -> forward -> receive -> restore (longest match first)
9. Streaming response restoration: buffer to token boundary before replacement
10. Configuration: whitelist, confidence threshold, PII type toggles
11. Transparent injection as Claude Agent SDK hooks

### Phase 3: Memory System
12. SQLite adapter (better-sqlite3, zero external dependencies)
13. Unified embedding interface: local-first (transformers.js + all-MiniLM-L6-v2 ONNX), optional cloud API
14. Memory Blocks implementation (SQL storage, version optimistic lock, readOnly validation)
15. memory_replace / memory_insert tools (agent self-editing Core Memory)
16. Recall Memory (dual-write SQL + sqlite-vec Vector, async embedding)
17. Hybrid retrieval engine (Vector + BM25/FTS5 + Reciprocal Rank Fusion)
18. Namespace-based ACL (full unit test coverage)
19. Provenance tracking

### Phase 4: Orchestration Engine
20. `delegate` MCP tool (single-layer query() -> validate route -> return result)
21. Orchestrator (multi-layer delegate chains, session management, concurrency control)
22. ContextBuilder (Write/Select/Compress/Isolate pipeline)
23. `delegateParallel` + result compression
24. Escalation + human-approval flow
25. EventBus (SubagentStart/Stop events -> dashboard push)

### Phase 5: Semantic Memory + Consolidation
26. Fact extraction pipeline (Mem0-style 5 stages: extract -> embed -> conflict detect -> LLM judge -> write)
27. Deduplication: content hash + semantic threshold + LLM ADD/UPDATE/DELETE
28. Time awareness: bi-temporal axis (valid_time + recorded_time, inspired by Zep)
29. Graph Memory adapter (Neo4j optional, default SQLite relational table simplified version)
30. Sleep-time consolidator (background worker)

### Phase 6: Communication Integration
31. Gateway (Chat SDK onNewMention/onSubscribedMessage/onAction)
32. Stream bridge (query() AsyncGenerator -> AsyncIterable<string>)
33. JSX Card components (status, approval, agent tree)
34. Webhook server (Next.js API routes)

### Phase 7: Web UI Dashboard
35. Project setup (Next.js + shadcn + BaseUI)
36. Dashboard overview page
37. Agent Tree (React Flow interactive hierarchy graph)
38. Task Flow real-time monitoring (WebSocket + flow diagram)
39. Memory Viewer (Core editor + Recall search + Semantic list + Graph visualization)
40. Config Editor (YAML + schema validation + live preview)
41. Logs aggregation page
42. Privacy Dashboard (masking statistics, Vault status, detection logs)

### Phase 8: Production Hardening
43. Postgres adapter (pgvector + FTS)
44. Neo4j adapter (full graph)
45. Docker Compose development environment
46. Documentation and example configurations
47. CI/CD + release pipeline

---

## 11. Verification Plan

1. **Privacy Gateway Tests**: PII detection accuracy (covering all types), Vault consistency (same mapping within session), streaming restoration correctness, whitelist exclusion, code block conservative strategy
2. **ACL Unit Tests**: Full combinatorial coverage of permission matrix (self/superior/subordinate/sibling x read/write x each tier)
3. **Memory Integration Tests**: block self-edit -> recall dual-write + hybrid retrieval -> fact extraction (ADD/UPDATE/DELETE) -> dedup -> consolidation -> temporal axis queries
4. **Orchestration Tests**: 3-layer delegate chain (CEO -> VP -> Dev), message passing + result compression propagation + parallel delegation
5. **Privacy + Orchestration E2E**: User message containing PII -> mask -> 3-layer agent processing -> restore -> verify real data does not appear in LLM API logs
6. **Pure Local Tests**: Offline environment + Ollama + SQLite, verify full pipeline runs offline
7. **Dashboard**: Launch -> trigger task -> verify real-time updates / memory viewing / config editing / privacy statistics
