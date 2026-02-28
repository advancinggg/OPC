# OPC Product PRD Workbook

Status: Draft (fill-in workbook)
Owner: [TBD by Product Owner]
Last Updated: [TBD]
Document Version: 0.1

## How To Use This Workbook
1. Fill every `[TBD]` field in order.
2. Keep requirement IDs stable once created (`REQ-*`, `NFR-*`).
3. Write acceptance criteria in testable terms.
4. Treat this document as the source of truth for scope.
5. Architecture and implementation documents must reference these IDs.

## 1. Product Context
1. Product one-liner: [TBD]
2. Primary users: [TBD]
3. Secondary users: [TBD]
4. Core problem statement: [TBD]
5. Why now: [TBD]
6. Non-goals: [TBD]

## 2. Outcomes And Success Metrics
1. North-star metric: [TBD]
2. Supporting metrics (3-5): [TBD]
3. Current baseline per metric: [TBD]
4. Target per metric: [TBD]
5. Target date per metric: [TBD]
6. Failure criteria (how to judge this release failed): [TBD]

## 3. Users, JTBD, And Primary Scenarios
### 3.1 Personas
1. Persona A: [TBD]
2. Persona B: [TBD]
3. Persona C: [TBD]

### 3.2 Jobs To Be Done
1. Job 1: [TBD]
2. Job 2: [TBD]
3. Job 3: [TBD]

### 3.3 Top Scenarios (MVP Candidate List)
1. Scenario name: [TBD]
Trigger: [TBD]
Expected output: [TBD]
Constraints: [TBD]

2. Scenario name: [TBD]
Trigger: [TBD]
Expected output: [TBD]
Constraints: [TBD]

3. Scenario name: [TBD]
Trigger: [TBD]
Expected output: [TBD]
Constraints: [TBD]

## 4. Scope Definition
### 4.1 MVP In Scope
1. [TBD]
2. [TBD]
3. [TBD]

### 4.2 Out Of Scope (Current Release)
1. [TBD]
2. [TBD]
3. [TBD]

## 5. Functional Requirements (REQ-*)
Use this template for each requirement:
- Requirement: clear behavior
- Rationale: why it matters
- Acceptance criteria: testable, observable
- Priority: P0/P1/P2
- MVP: Yes/No

### Requirement List
REQ-001
Requirement: Root-agent conversation flow must support [TBD].
Rationale: [TBD].
Acceptance criteria: [TBD].
Priority: [TBD].
MVP: [TBD].

REQ-002
Requirement: Hierarchical delegation must support [TBD].
Rationale: [TBD].
Acceptance criteria: [TBD].
Priority: [TBD].
MVP: [TBD].

REQ-003
Requirement: Session lifecycle must support create, resume, and recover for [TBD].
Rationale: [TBD].
Acceptance criteria: [TBD].
Priority: [TBD].
MVP: [TBD].

REQ-004
Requirement: Tool execution policy must support [TBD].
Rationale: [TBD].
Acceptance criteria: [TBD].
Priority: [TBD].
MVP: [TBD].

REQ-005
Requirement: Model routing must support provider/model portability for [TBD].
Rationale: [TBD].
Acceptance criteria: [TBD].
Priority: [TBD].
MVP: [TBD].

REQ-006
Requirement: Memory write/read behavior must support [TBD].
Rationale: [TBD].
Acceptance criteria: [TBD].
Priority: [TBD].
MVP: [TBD].

REQ-007
Requirement: Human approval and escalation flow must support [TBD].
Rationale: [TBD].
Acceptance criteria: [TBD].
Priority: [TBD].
MVP: [TBD].

REQ-008
Requirement: Channel support rollout order is Telegram -> Discord -> Slack.
Rationale: [TBD].
Acceptance criteria: [TBD].
Priority: [TBD].
MVP: [TBD].

REQ-009
Requirement: Runtime observability must include [TBD].
Rationale: [TBD].
Acceptance criteria: [TBD].
Priority: [TBD].
MVP: [TBD].

REQ-010
Requirement: Admin/config workflow must support [TBD].
Rationale: [TBD].
Acceptance criteria: [TBD].
Priority: [TBD].
MVP: [TBD].

## 6. Local-Native Agent Requirements (Coverage Checklist)
### 6.1 Runtime Environment
1. Supported OS targets: [TBD]
2. Required runtime versions: [TBD]
3. CPU/GPU constraints: [TBD]
4. Memory/disk budget per instance: [TBD]
5. Multi-project isolation model: [TBD]

### 6.2 Local Security And Permissions
1. File access boundary policy: [TBD]
2. Command execution policy: [TBD]
3. Network egress policy: [TBD]
4. Secret storage and rotation: [TBD]
5. Audit requirements for local actions: [TBD]

### 6.3 Reliability And Recovery
1. Crash recovery expectation: [TBD]
2. Resume semantics after restart: [TBD]
3. Idempotency expectations for tool calls: [TBD]
4. Deterministic replay/debug requirements: [TBD]
5. Backup/restore requirements: [TBD]

### 6.4 Operations On Developer Machines
1. Install/bootstrap requirements: [TBD]
2. Upgrade/migration policy: [TBD]
3. Uninstall/cleanup policy: [TBD]
4. Diagnostics collection policy: [TBD]

## 7. Memory Requirements (Database vs File System Decision)
### 7.1 Memory Type Decision Matrix
| Memory Type | Example Data | Read/Write Pattern | Latency Need | Human Editable | FS-only sufficient? | Indexed persistent store needed? | Retention | Notes |
|---|---|---|---|---|---|---|---|---|
| Working memory | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| Episodic memory | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| Semantic memory | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| Procedural memory | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| Core memory blocks | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |

### 7.2 Memory Policy Questions
1. What must be queryable by semantics/keywords? [TBD]
2. What must remain fully human-readable and hand-editable? [TBD]
3. What needs strict durability guarantees? [TBD]
4. What can be eventually consistent? [TBD]
5. What must support provenance and audit trails? [TBD]
6. What data can be dropped or compacted? [TBD]
7. What privacy rules apply per memory type? [TBD]
8. What import/export portability is required? [TBD]

## 8. Non-Functional Requirements (NFR-*)
NFR-001 Performance targets (latency/throughput): [TBD]
NFR-002 Availability and reliability targets: [TBD]
NFR-003 Recovery objectives (RTO/RPO): [TBD]
NFR-004 Security requirements: [TBD]
NFR-005 Privacy/compliance constraints: [TBD]
NFR-006 Cost constraints (local/cloud/hybrid): [TBD]
NFR-007 Scalability boundaries: [TBD]
NFR-008 Observability and diagnostics: [TBD]
NFR-009 Developer experience requirements: [TBD]
NFR-010 Backward compatibility requirements: [TBD]

## 9. Architecture Input Section (To Be Consumed By architecture.md)
1. Runtime orchestration constraints: [TBD]
2. Model router constraints: [TBD]
3. Memory backend constraints: [TBD]
4. Channel adapter constraints: [TBD]
5. Privacy gateway constraints: [TBD]
6. Event contract constraints: [TBD]
7. Deployment constraints: [TBD]

## 10. Implementation Input Section (To Be Consumed By implementation-plan.md)
1. MVP release date target: [TBD]
2. Team capacity assumptions: [TBD]
3. Sequence constraints: [TBD]
4. Phase gate test requirements: [TBD]
5. Rollout strategy constraints: [TBD]
6. Rollback requirements: [TBD]
7. Operational readiness requirements: [TBD]

## 11. Risks, Open Questions, And Assumptions
### 11.1 Open Questions
1. [TBD]
2. [TBD]
3. [TBD]

### 11.2 Product Assumptions
1. [TBD]
2. [TBD]

### 11.3 Technical Assumptions
1. V1 implementation is TypeScript-only.
2. Rust implementation is out of scope for this repository.
3. Channel rollout order is Telegram -> Discord -> Slack.
4. Vercel Chat SDK is optional, not a required core dependency.
5. Additional assumptions: [TBD]

## 12. Traceability Seed Table
| Requirement ID | Architecture Section | Implementation Item IDs | Test IDs | Status |
|---|---|---|---|---|
| REQ-001 | [TBD] | [TBD] | [TBD] | [TBD] |
| REQ-002 | [TBD] | [TBD] | [TBD] | [TBD] |
| REQ-003 | [TBD] | [TBD] | [TBD] | [TBD] |
| REQ-004 | [TBD] | [TBD] | [TBD] | [TBD] |
| REQ-005 | [TBD] | [TBD] | [TBD] | [TBD] |
| REQ-006 | [TBD] | [TBD] | [TBD] | [TBD] |
| REQ-007 | [TBD] | [TBD] | [TBD] | [TBD] |
| REQ-008 | [TBD] | [TBD] | [TBD] | [TBD] |
| REQ-009 | [TBD] | [TBD] | [TBD] | [TBD] |
| REQ-010 | [TBD] | [TBD] | [TBD] | [TBD] |
| NFR-001 | [TBD] | [TBD] | [TBD] | [TBD] |
| NFR-002 | [TBD] | [TBD] | [TBD] | [TBD] |
| NFR-003 | [TBD] | [TBD] | [TBD] | [TBD] |
| NFR-004 | [TBD] | [TBD] | [TBD] | [TBD] |
| NFR-005 | [TBD] | [TBD] | [TBD] | [TBD] |
| NFR-006 | [TBD] | [TBD] | [TBD] | [TBD] |
| NFR-007 | [TBD] | [TBD] | [TBD] | [TBD] |
| NFR-008 | [TBD] | [TBD] | [TBD] | [TBD] |
| NFR-009 | [TBD] | [TBD] | [TBD] | [TBD] |
| NFR-010 | [TBD] | [TBD] | [TBD] | [TBD] |
