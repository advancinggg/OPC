# OPC Implementation Plan

Status: Draft template
Owner: [TBD]
Last Updated: [TBD]
Document Version: 0.1

Source documents:
1. `docs/product-prd.md`
2. `docs/architecture.md`

## 1. Planning Rules
1. Every implementation item must map to `REQ/NFR` and `ADR`.
2. No implementation item without acceptance criteria.
3. Every phase has explicit entry and exit gates.
4. Every phase has required tests (`TST-*`).

## 2. Delivery Strategy
1. Delivery model: iterative milestones with gated validation.
2. Default sequencing:
- Phase A: Contracts and foundations
- Phase B: Runtime MVP + Telegram
- Phase C: Durability and recovery
- Phase D: Memory and privacy depth
- Phase E: Discord and Slack rollout
- Phase F: Hardening and release readiness

Constraints from PRD: [TBD]

## 3. Work Item Template (IMP-*)
IMP-ID: [TBD]
Title: [TBD]
Description: [TBD]
Linked REQ/NFR: [TBD]
Linked ADR: [TBD]
Dependencies: [TBD]
Deliverables: [TBD]
Acceptance criteria: [TBD]
Owner: [TBD]
Target milestone: [TBD]
Status: [TBD]

## 4. Phases And Milestones
## Phase A - Contracts And Project Foundations
Entry criteria:
1. PRD requirements version frozen for this milestone.
2. Architecture ADRs needed for Phase A are proposed.

Implementation items:
1. IMP-001 [TBD]
2. IMP-002 [TBD]
3. IMP-003 [TBD]

Required tests:
1. TST-001 [TBD]
2. TST-002 [TBD]

Exit criteria:
1. Core contracts defined and reviewed.
2. Traceability links populated for Phase A items.

## Phase B - Runtime MVP + Telegram
Entry criteria:
1. Phase A exit criteria met.
2. REQ-001/REQ-002/REQ-008 acceptance drafts approved.

Implementation items:
1. IMP-101 [TBD]
2. IMP-102 [TBD]
3. IMP-103 [TBD]
4. IMP-104 [TBD]

Required tests:
1. TST-101 [TBD]
2. TST-102 [TBD]
3. TST-103 [TBD]

Exit criteria:
1. Runtime handles root conversation and delegation.
2. Telegram integration passes MVP acceptance criteria.

## Phase C - Durability And Recovery
Entry criteria:
1. Phase B exit criteria met.
2. Recovery objectives from NFR are finalized.

Implementation items:
1. IMP-201 [TBD]
2. IMP-202 [TBD]
3. IMP-203 [TBD]

Required tests:
1. TST-201 [TBD]
2. TST-202 [TBD]
3. TST-203 [TBD]

Exit criteria:
1. Crash/restart recovery works per PRD acceptance.
2. Idempotency guarantees validated for tool execution.

## Phase D - Memory + Privacy Depth
Entry criteria:
1. Phase C exit criteria met.
2. Memory matrix decisions in PRD are filled.

Implementation items:
1. IMP-301 [TBD]
2. IMP-302 [TBD]
3. IMP-303 [TBD]
4. IMP-304 [TBD]

Required tests:
1. TST-301 [TBD]
2. TST-302 [TBD]
3. TST-303 [TBD]

Exit criteria:
1. Memory behavior matches PRD decisions.
2. Privacy masking and restoration pass acceptance criteria.

## Phase E - Channel Expansion (Discord -> Slack)
Entry criteria:
1. Phase D exit criteria met.
2. Channel behavior parity criteria defined.

Implementation items:
1. IMP-401 [TBD]
2. IMP-402 [TBD]
3. IMP-403 [TBD]

Required tests:
1. TST-401 [TBD]
2. TST-402 [TBD]

Exit criteria:
1. Discord integration meets acceptance criteria.
2. Slack integration meets acceptance criteria.
3. Cross-channel behavior parity validated.

## Phase F - Hardening And Release Readiness
Entry criteria:
1. Phase E exit criteria met.
2. NFR coverage checklist complete.

Implementation items:
1. IMP-501 [TBD]
2. IMP-502 [TBD]
3. IMP-503 [TBD]

Required tests:
1. TST-501 [TBD]
2. TST-502 [TBD]
3. TST-503 [TBD]

Exit criteria:
1. SLO/SLA readiness approved.
2. Observability and incident runbooks approved.
3. Release checklist signed off.

## 5. Dependency Map
| Item ID | Depends On | Blocks | Notes |
|---|---|---|---|
| IMP-001 | [TBD] | [TBD] | [TBD] |
| IMP-101 | [TBD] | [TBD] | [TBD] |
| IMP-201 | [TBD] | [TBD] | [TBD] |
| IMP-301 | [TBD] | [TBD] | [TBD] |
| IMP-401 | [TBD] | [TBD] | [TBD] |
| IMP-501 | [TBD] | [TBD] | [TBD] |

## 6. Test Strategy (TST-*)
1. Unit test policy: [TBD]
2. Integration test policy: [TBD]
3. E2E test policy: [TBD]
4. Failure-injection test policy: [TBD]
5. Regression gate policy: [TBD]

### Test Case Template
TST-ID: [TBD]
Name: [TBD]
Validates: [REQ/NFR/IMP IDs]
Setup: [TBD]
Steps: [TBD]
Expected result: [TBD]
Pass/fail evidence: [TBD]

## 7. Rollout, Monitoring, And Incident Readiness
1. Rollout strategy: [TBD]
2. Rollback strategy: [TBD]
3. Monitoring dashboards required: [TBD]
4. Alerting policy: [TBD]
5. Incident response owner and SLA: [TBD]

## 8. Risk Register
| Risk ID | Description | Impact | Likelihood | Mitigation | Owner | Status |
|---|---|---|---|---|---|---|
| RSK-001 | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| RSK-002 | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |
| RSK-003 | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] | [TBD] |

## 9. Traceability Matrix
| PRD ID | ADR ID(s) | IMP ID(s) | TST ID(s) | Coverage Status |
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

## 10. Completion Checklist
1. Every REQ/NFR has at least one IMP and TST link.
2. Every IMP has explicit acceptance criteria.
3. Every phase has pass/fail gates.
4. Rollback and incident plan are defined before release.
5. Remaining open questions are tracked with owners and deadlines.
