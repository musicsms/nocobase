# Vulnerability Management Backend Production Hardening Specification

**Date:** 2026-05-09  
**Scope:** Backend-only production hardening for VM subsystem (pre-frontend expansion)  
**Status:** Draft for approval

---

## 1. Purpose

Define production-grade backend requirements to evolve VM from Phase 1 MVP into a durable, DB-backed, auditable, and operable service layer.

This specification is the source of truth for Phase 2 backend implementation planning and execution.

---

## 2. Non-Goals

- No frontend UX expansion or new UI pages in this phase.
- No bidirectional external ticketing sync.
- No immutable signed audit ledger (deferred to future phase).

---

## 3. Baseline and Problem Statement

Phase 1 delivered functional MVP logic with broad tests, but production-readiness gaps remain:
- Runtime business paths still include in-memory persistence patterns.
- Transaction boundaries are not formalized for cross-collection mutations.
- Audit event coverage is incomplete vs required minimum set.
- Policy consistency drift exists (calendar vs business-day semantics).
- Performance harness is not representative of DB-backed runtime.

---

## 4. Functional Requirements

### 4.1 Persistence and Data Integrity

1. All persistent VM business flows MUST use DB-backed repositories.
2. In-memory arrays MAY be used only in isolated unit tests, never in runtime service paths.
3. Cross-collection write flows MUST execute in a single transaction context.
4. On failure, transaction rollback MUST prevent partial side effects.

Required repositories:
- vm_findings
- vm_finding_instances
- vm_ingestion_jobs
- vm_ingestion_rejects
- vm_exceptions
- vm_remediation_tasks
- vm_activity_logs

Fingerprint storage note:
- Fingerprint identity is currently embedded in `vm_findings.fingerprint` (no separate `vm_fingerprints` collection in current scope).

Migration requirement:
- Schema migrations required for new constraints and indexes MUST be dry-run validated and rollback-tested before deployment.

### 4.2 Transaction Model (NocoBase/Sequelize)

1. Repository access MUST use NocoBase repository APIs (`app.db.getRepository(...)` and collection-scoped repositories).
2. Cross-collection orchestration MUST use explicit `app.db.sequelize.transaction(...)`.
3. Transaction context MUST be propagated to every repository call in a flow.
4. Nested transaction behavior MUST be documented and covered by contract tests.

### 4.3 Lifecycle/ACL Correctness

1. Rejected-exception recovery path MUST enforce explicit role boundaries:
   - `vm_admin` and scoped `vm_analyst` MUST be permitted to perform `accepted_risk -> triaged`.
   - `vm_triage_queue` MUST NOT be permitted to perform `accepted_risk -> triaged`.
2. `vm_triage_queue` permissions MUST be explicitly validated for allowed/disallowed recovery edges.
3. Transition legality and ACL authorization MUST remain independently testable.

### 4.4 Risk/SLA/KEV Policy Consistency

1. Due-date computation and escalation checks MUST use one canonical day-count policy based on business days in organization-configured timezone and holiday calendar.
2. KEV retroactive re-due MUST follow the same day-count semantics as normal SLA computations.
3. KEV tightening rule MUST remain `min(3 days, existing policy due date)` (earlier deadline wins).
4. KPI semantics MUST explicitly define closed-like states treatment.
5. `0-row` ingestion outcome MUST use explicit, non-misleading status semantics.

### 4.5 Correlation and Source Robustness

1. Correlation keys MUST remain deterministic and version-stable.
2. Correlation key MUST NOT include scanner-version-dependent fields (for example raw title without normalization) that prevent merging the same vulnerability across tool version upgrades.
3. Correlation strategy MUST document fallback behavior and confidence impact.
4. Last-resort title-based fallback is permitted only as normalized `title_hash` with explicit low-confidence labeling, and such records MUST be routed to analyst review queue.
5. Source-type inference MUST NOT depend on fragile filename patterns only.

### 4.6 Audit Completeness

Minimum audit event set:
- `state_changed`
- `assignment_changed`
- `risk_override`
- `asset_scope_changed`
- `exception_requested`
- `exception_approved`
- `exception_rejected`
- `exception_revoked`
- `due_date_changed`

Audit event requirements:
- actor identity
- timestamp
- entity id/type
- old/new values (or diff payload)
- reason/comment where required
- correlation/job id where applicable

### 4.7 Asset Scope Snapshot Integrity

1. Scope-affecting updates MUST preserve pre-update scope snapshot before overwrite.
2. Scope changes MUST be auditable (`asset_scope_changed`) when they affect policy outcomes.
   - At minimum, scope changes that modify organization assignment, team ownership, or SLA tier MUST produce an `asset_scope_changed` audit record.
3. Snapshot/diff semantics MUST be deterministic for re-detection and review.

### 4.8 Connector Runtime Hardening

1. Connector registry MUST support configurable runtime registration (not hardcoded single connector path).
2. Connector output contract MUST align with normalized ingestion schema used by ingestion pipeline.
3. Connector sync jobs MUST support retry/idempotency and error taxonomy.

---

## 5. Non-Functional Requirements

### 5.1 Performance SLOs

Under target profile (4 vCPU, 8 GB RAM app node, PostgreSQL 14+):
- Bulk ingest 50,000 finding instances: target <= 120s (stretch <= 60s).
- Finding transition operation latency: p95 <= 200ms, p99 <= 500ms under nominal analyst load.
- Escalation scan for 10,000 open findings: <= 30s per run.

SLO evidence requirement:
- Performance tests MUST be executed on target profile (4 vCPU / 8 GB RAM / PostgreSQL 14+) or a documented equivalent.
- Results from developer-local environments are not acceptable as production SLO evidence.

### 5.2 Reliability

- No orphan records after failed multi-step workflows.
- Retry-safe behavior for idempotent ingestion and connector sync.
- Concurrency safety for parallel ingestion, transitions, and KEV updates.

### 5.3 Operability

Required observability:
- ingestion latency and throughput
- reject rate and top rejection causes
- escalation counts and overdue volume
- queue depth by state/team

Runbooks MUST include:
- operations routine
- incident response
- rollback steps
- verification checklist

---

## 6. Verification Requirements

Required test layers:
1. Unit tests for pure policy logic.
2. DB integration tests for repository + transaction behavior.
3. Workflow contract tests for lifecycle/escalation/KEV.
4. Failure-path tests for partial import, strict mode, retry, rollback.
5. Performance tests on DB-backed runtime profile.

Completion claims MUST include executable evidence (test command outputs and measured results).

---

## 7. Rollout Gates (Backend)

All gates MUST pass before frontend implementation starts:
1. Phase 1 baseline frozen (committed + tagged).
2. Migration dry-run + rollback drill pass.
3. DB-backed end-to-end backend flow pass:
   - import -> correlate -> risk/sla -> lifecycle -> audit.
4. Audit completeness validated against required event set.
5. Performance SLOs met, or exceptions approved with remediation commitments.
6. Operations/security sign-off on runbooks and alerts.

---

## 8. Known Accepted Limitations (Current Decision)

- `fqdn` fallback to `row.host` may contain IP values; accepted temporarily unless downstream reporting demands strict FQDN semantics.

---

## 9. Traceability to Plan

Implementation plan MUST map each sprint/task to these requirement sections with explicit acceptance criteria linkage.
