# OPC Architecture Specification

Status: Draft template
Owner: [TBD]
Last Updated: [TBD]
Document Version: 0.1

Source of truth: `docs/product-prd.md`

## 1. Purpose And Scope
1. Purpose: define technical architecture that satisfies PRD requirements.
2. In scope: technical decisions and contracts for V1 TypeScript implementation.
3. Out of scope: product prioritization and milestone scheduling details.

## 2. Input Requirements Snapshot
List all linked requirements from PRD before making decisions.

| PRD ID | Summary | Decision Needed? | Covered By |
|---|---|---|---|
| REQ-001 | [TBD] | [TBD] | [TBD] |
| REQ-002 | [TBD] | [TBD] | [TBD] |
| REQ-003 | [TBD] | [TBD] | [TBD] |
| REQ-004 | [TBD] | [TBD] | [TBD] |
| REQ-005 | [TBD] | [TBD] | [TBD] |
| REQ-006 | [TBD] | [TBD] | [TBD] |
| REQ-007 | [TBD] | [TBD] | [TBD] |
| REQ-008 | [TBD] | [TBD] | [TBD] |
| REQ-009 | [TBD] | [TBD] | [TBD] |
| REQ-010 | [TBD] | [TBD] | [TBD] |
| NFR-001 | [TBD] | [TBD] | [TBD] |
| NFR-002 | [TBD] | [TBD] | [TBD] |
| NFR-003 | [TBD] | [TBD] | [TBD] |
| NFR-004 | [TBD] | [TBD] | [TBD] |
| NFR-005 | [TBD] | [TBD] | [TBD] |
| NFR-006 | [TBD] | [TBD] | [TBD] |
| NFR-007 | [TBD] | [TBD] | [TBD] |
| NFR-008 | [TBD] | [TBD] | [TBD] |
| NFR-009 | [TBD] | [TBD] | [TBD] |
| NFR-010 | [TBD] | [TBD] | [TBD] |

## 3. System Context And Boundaries
1. External systems: [TBD]
2. User touchpoints: [TBD]
3. Trust boundaries: [TBD]
4. Deployment topology: [TBD]

## 4. High-Level Components
| Component | Responsibility | Key Interfaces | Data Owned |
|---|---|---|---|
| `@opc/runtime-kernel` | Orchestration, delegation, session lifecycle | `RuntimeKernel` | Session state, task state |
| `@opc/model-router` | LLM provider abstraction and policy routing | `ModelAdapter` | Provider config, model policy |
| `@opc/memory` | Memory storage/retrieval/recovery | `MemoryBus` | events, checkpoints, facts |
| `@opc/privacy` | PII masking/unmasking middleware | `PrivacyMiddleware` | vault mappings, policies |
| `@opc/channel-gateway` | Channel event normalization and outbound delivery | `ChannelAdapter` | channel session mapping |
| `@opc/control-plane` | config validation, observability, admin APIs | `ControlAPI` | configs, diagnostics metadata |

## 5. Runtime And Data Flows
### 5.1 Single Task Flow
1. Channel event received by adapter.
2. Gateway normalizes event and invokes runtime.
3. Runtime builds context and calls model-router.
4. Tool calls execute via policy guard.
5. Results are streamed and persisted.

Open decisions: [TBD]

### 5.2 Delegation Flow
1. Parent agent emits `delegate` action.
2. Runtime validates hierarchy and policy.
3. Child session is created with isolated context.
4. Child result is returned and optionally compressed.
5. Parent session resumes.

Open decisions: [TBD]

### 5.3 Recovery Flow
1. Runtime restart detected.
2. Checkpoint lookup by session.
3. If missing, replay from event log.
4. Resume suspended task.

Open decisions: [TBD]

## 6. Memory Architecture
### 6.1 Memory Layers
1. Working memory: [TBD]
2. Episodic memory: [TBD]
3. Semantic memory: [TBD]
4. Procedural memory: [TBD]
5. Core memory blocks: [TBD]

### 6.2 Persistence Strategy
1. File-system backed memory scope: [TBD]
2. Indexed/persistent store scope: [TBD]
3. Consistency model per memory type: [TBD]
4. Retention and compaction: [TBD]
5. Provenance and audit metadata: [TBD]

### 6.3 Durability Contract
1. Event append contract: [TBD]
2. Checkpoint contract: [TBD]
3. Resume/replay contract: [TBD]
4. Idempotency contract for side-effecting tools: [TBD]

## 7. Public Interfaces And Contracts
```ts
export interface RuntimeKernel {
  execute(task: RootTask): Promise<TaskResult>;
  delegate(input: DelegateInput): Promise<DelegateResult>;
  resume(sessionId: string): Promise<void>;
}

export interface ModelAdapter {
  id: string;
  supportsTools: boolean;
  supportsStreaming: boolean;
  invoke(req: ModelRequest): Promise<ModelResponse>;
  stream(req: ModelRequest): AsyncIterable<ModelChunk>;
}

export interface MemoryBus {
  appendEvent(evt: RuntimeEvent): Promise<void>;
  checkpoint(snapshot: SessionSnapshot): Promise<void>;
  recover(sessionId: string): Promise<SessionSnapshot | null>;
}

export interface ChannelAdapter {
  platform: "telegram" | "discord" | "slack";
  start(handler: ChannelEventHandler): Promise<void>;
  send(message: OutboundMessage): Promise<void>;
  ack(eventId: string): Promise<void>;
}
```

Contract decisions to lock:
1. Runtime event schema: [TBD]
2. Versioning policy: [TBD]
3. Backward compatibility policy: [TBD]

## 8. Security, Privacy, And Compliance
1. PII detection and masking boundary: [TBD]
2. Secret handling design: [TBD]
3. Access control model: [TBD]
4. Redaction policy in logs/events: [TBD]
5. Compliance constraints from PRD: [TBD]

## 9. Performance And Scalability
1. Throughput target mapping (NFR-001): [TBD]
2. Latency budget by stage: [TBD]
3. Concurrency control strategy: [TBD]
4. Bottleneck mitigation plan: [TBD]

## 10. Observability And Operability
1. Required events and metrics: [TBD]
2. Trace strategy: [TBD]
3. Alert thresholds: [TBD]
4. Debug/replay tooling: [TBD]

## 11. Architecture Decision Records (ADR-*)
Use one ADR block per decision.

### ADR-001 Runtime Kernel Independence
Status: Proposed
Decision: Runtime must not depend on vendor agent SDKs.
Rationale: [TBD]
Linked PRD IDs: REQ-001, REQ-002, REQ-003, REQ-005
Consequences: [TBD]
Alternatives considered: [TBD]

### ADR-002 Channel Adapter Abstraction
Status: Proposed
Decision: Channel integration uses adapters, Telegram first.
Rationale: [TBD]
Linked PRD IDs: REQ-008
Consequences: [TBD]
Alternatives considered: [TBD]

### ADR-003 Memory Backends As Policy
Status: Proposed
Decision: Memory contracts are core; storage backends are pluggable.
Rationale: [TBD]
Linked PRD IDs: REQ-006, NFR-002, NFR-003
Consequences: [TBD]
Alternatives considered: [TBD]

### ADR-004 Vercel Chat SDK Optionality
Status: Proposed
Decision: Vercel Chat SDK is optional and not a core dependency.
Rationale: [TBD]
Linked PRD IDs: REQ-008
Consequences: [TBD]
Alternatives considered: [TBD]

### ADR-005 TypeScript-Only For This Repository
Status: Accepted
Decision: V1 in this repository is TypeScript-only.
Rationale: product/team constraint
Linked PRD IDs: REQ-010, NFR-009
Consequences: [TBD]
Alternatives considered: [TBD]

## 12. Risks And Open Issues
1. Technical risk: [TBD]
Mitigation: [TBD]

2. Operational risk: [TBD]
Mitigation: [TBD]

3. Product risk: [TBD]
Mitigation: [TBD]

## 13. Traceability Table
| PRD ID | ADR IDs | Architecture Section | Status |
|---|---|---|---|
| REQ-001 | [TBD] | [TBD] | [TBD] |
| REQ-002 | [TBD] | [TBD] | [TBD] |
| REQ-003 | [TBD] | [TBD] | [TBD] |
| REQ-004 | [TBD] | [TBD] | [TBD] |
| REQ-005 | [TBD] | [TBD] | [TBD] |
| REQ-006 | [TBD] | [TBD] | [TBD] |
| REQ-007 | [TBD] | [TBD] | [TBD] |
| REQ-008 | [TBD] | [TBD] | [TBD] |
| REQ-009 | [TBD] | [TBD] | [TBD] |
| REQ-010 | [TBD] | [TBD] | [TBD] |
| NFR-001 | [TBD] | [TBD] | [TBD] |
| NFR-002 | [TBD] | [TBD] | [TBD] |
| NFR-003 | [TBD] | [TBD] | [TBD] |
| NFR-004 | [TBD] | [TBD] | [TBD] |
| NFR-005 | [TBD] | [TBD] | [TBD] |
| NFR-006 | [TBD] | [TBD] | [TBD] |
| NFR-007 | [TBD] | [TBD] | [TBD] |
| NFR-008 | [TBD] | [TBD] | [TBD] |
| NFR-009 | [TBD] | [TBD] | [TBD] |
| NFR-010 | [TBD] | [TBD] | [TBD] |
