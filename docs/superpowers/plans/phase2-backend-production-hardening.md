# Vulnerability Management Backend Production Hardening Plan

**Date:** 2026-05-09  
**Owner:** VM Engineering  
**Status:** Draft for review  
**Prerequisite:** Phase 1 MVP implementation completed  
**Derived from spec:** `docs/superpowers/specs/2026-05-09-vm-backend-production-hardening-spec.md`

---

## 1. Objective

Harden backend of Vulnerability Management (VM) from MVP-level implementation to production-ready service quality **before** any frontend/runtime UI expansion.

Primary outcomes:
- Eliminate in-memory business-path dependencies in runtime flows.
- Enforce DB-backed consistency, transaction safety, and auditability.
- Align policy behavior (risk/SLA/KEV/lifecycle) across all services.
- Validate performance and reliability under production-like profile.

---

## 2. Scope

### In Scope
- Server plugin runtime logic and data persistence layer.
- Ingestion, normalization, correlation, lifecycle transitions, SLA escalation, KEV retroactive updates.
- Connector framework runtime hardening and first connector productionization.
- Audit event completeness and persistence.
- DB-backed test strategy, performance benchmarking, and rollout readiness.

### Out of Scope
- Frontend UX development and dashboard/page wiring expansion.
- External ticketing bi-directional sync (Jira/ServiceNow).
- Immutable cryptographic audit stream (planned for later phase).

---

## 3. Current State Summary (Baseline)

Phase 1 achieved broad MVP coverage with unit-style behavior tests and partial integration logic.  
Remaining hardening gaps:
- Core business paths still rely on in-memory data structures.
- Policy inconsistencies remain (notably date semantics across SLA/KEV/escalation).
- Audit event surface is incomplete vs design minimum set.
- Connector runtime is framework-complete but not fully operationalized in DB-backed ingestion path.
- Performance verification is in-memory-biased, not DB-throughput representative.

Pre-execution note:
- Current workspace still contains uncommitted/untracked Phase 1 artifacts. Phase 2 must not start until Phase 1 baseline is committed and tagged.

---

## 4. Target Production Architecture (Backend)

1. **Repository Layer (DB-backed):** Explicit repositories for `vm_findings`, `vm_finding_instances`, `vm_ingestion_jobs`, `vm_ingestion_rejects`, `vm_exceptions`, `vm_remediation_tasks`, `vm_activity_logs`.
2. **Application Services:** Stateless services orchestrating repositories via transaction boundaries.
3. **Policy Engine:** Centralized risk/SLA/calendar/lifecycle rules to avoid drift across workflows.
4. **Workflow/Audit Pipeline:** Deterministic side-effects: transition -> state persist -> audit persist -> notification enqueue.
5. **Observability Layer:** Structured logs, job metrics, rejection taxonomy, and escalation counters.

---

## 5. Work Plan (Backend-first)

## Task 0 (Mandatory): Phase 1 Baseline Freeze

### Goals
- Protect completed Phase 1 implementation from loss/regression before hardening starts.

### Tasks
- Stage and commit all Phase 1 files (including currently untracked plugin/runbook/test artifacts).
- Create a baseline tag (example: `vm-phase1-baseline`) and attach test evidence.
- Record Phase 1 open-item list in release notes as explicit carry-over debt.

### Acceptance Criteria
- `git status` is clean for baseline commit scope.
- Baseline commit + tag exists and is shared with reviewers.
- Phase 2 work starts from this immutable baseline only.

## Sprint BH1: Persistence Refactor and Transaction Safety

### Goals
- Move runtime logic from in-memory collections to PostgreSQL-backed repositories.
- Establish transaction boundaries for all mutation-heavy flows.

### Tasks
- BH1 Spike: NocoBase transaction/repository integration design note:
  - Standardize usage of `app.db.getRepository(...)` and relation repositories.
  - Standardize cross-collection transaction boundary with `app.db.sequelize.transaction(...)`.
  - Document propagation rule: every repository call in a business flow must receive the same transaction context.
- Implement repositories for:
  - `vm_findings`
  - `vm_finding_instances`
  - `vm_ingestion_jobs`
  - `vm_ingestion_rejects`
  - `vm_exceptions`
  - `vm_remediation_tasks`
  - `vm_activity_logs`
- Refactor services to repository interfaces:
  - ingestion
  - fingerprint upsert
  - lifecycle transition
  - correlation apply
  - KEV enrichment
- Asset scope integrity requirement:
  - Persist pre-update `asset_scope` snapshot before overwrite in upsert/redetection paths when scope change impacts policy/audit.
- Add DB constraints/indexing:
  - fingerprint uniqueness strategy
  - due_date / state indexes for queue and escalation
  - CVE + open-state lookup for KEV updates
- Add/verify migrations for new indexes/constraints with dry-run + rollback validation.
- Introduce transaction wrappers and rollback-safe error handling.

### Acceptance Criteria
- No production path uses in-memory arrays for persistent entities.
- DB integration tests pass for ingest/upsert/transition/correlation/kev flows.
- Rollback behavior verified under simulated partial failure.
- Transaction propagation verified for multi-collection writes (single rollback cancels all side effects).

---

## Sprint BH2: Policy Consistency and Domain Integrity

### Goals
- Remove rule drift between services and workflows.
- Normalize semantics for SLA, KEV due dates, and KPI definitions.

### Tasks
- Unify calendar semantics:
  - Standardize due-date computation and escalation checks on same business-day service.
  - Canonical policy must use business days in organization-configured timezone and holiday calendar.
  - KEV tightening rule must remain `min(3 days, existing policy due date)` (earlier deadline wins).
- Finalize KPI semantics configuration:
  - Decide closed-like states (`false_positive`, `accepted_risk`) handling.
- Harden correlation policy:
  - Review key composition and add deterministic fallback handling.
  - Prohibit scanner-version-dependent raw-title keys.
  - Allow last-resort fallback only via normalized `title_hash` marked low-confidence and routed to analyst review queue.
- Correct ingestion edge semantics:
  - `0-row` import status and operator messaging.
- ACL recovery correctness fix:
  - Ensure exception-reject recovery path supports `accepted_risk -> triaged` with intended role model.
  - Validate `vm_triage_queue` behavior explicitly (allowed/disallowed edges) to prevent dead-end state ownership.

### Acceptance Criteria
- One canonical date-policy implementation consumed by SLA + KEV + escalation.
- Date-policy contract test verifies business-day + timezone + holiday behavior.
- KEV due-date recompute test verifies `min(3 days, existing policy due date)` rule.
- KPI outputs match documented domain policy.
- Regression tests added for all known policy edge cases.
- Correlation tests verify low-confidence `title_hash` fallback routing to analyst review queue.
- ACL transition tests cover recovery path for rejected exceptions and pass for intended roles.

---

## Sprint BH3: Audit Completeness and Compliance Hardening

### Goals
- Reach minimum auditable event coverage defined by design.
- Ensure audit writes are durable and tied to DB state transitions.

### Tasks
- Implement/extend audit event set:
  - `state_changed`
  - `assignment_changed`
  - `risk_override`
  - `asset_scope_changed`
  - `exception_requested`
  - `exception_approved`
  - `exception_rejected`
  - `exception_revoked`
  - `due_date_changed`
- Replace transient/singleton audit patterns with DB-injected writer.
- Enforce event payload standards:
  - actor
  - timestamp
  - old/new values
  - reason/comment
  - correlation/job id (where applicable)

### Acceptance Criteria
- Audit coverage tests pass for all lifecycle-critical transitions.
- Every mutating workflow produces expected audit records in DB.

---

## Sprint BH4: Connector and Ingestion Runtime Hardening

### Goals
- Productionize connector execution path and error handling.
- Ensure connector outputs align with ingestion normalization contract.

### Tasks
- Promote connector registry from hardcoded path to configurable runtime.
- Standardize connector output contracts against normalized ingestion schema.
- Add connector sync job states, retries, and rejection taxonomy.
- Harden source identity inference (avoid filename-fragile assumptions).

### Acceptance Criteria
- DefectDojo connector executes end-to-end to DB-backed ingest path.
- Retry/idempotency behavior verified under duplicate/partial failure scenarios.

---

## Sprint BH5: Performance, Reliability, and Operational Readiness

### Goals
- Validate backend under production-like workload and failure modes.
- Finalize operational readiness artifacts.

### Tasks
- Run DB-backed performance harness:
  - `50,000 finding instances` target profile.
- Concurrency tests:
  - parallel ingestion
  - transition races
  - KEV bulk updates during ingest
- Add observability instrumentation:
  - ingestion latency
  - reject rates
  - escalation counts
  - queue depth by state/team
- Complete runbook hardening:
  - operations
  - rollback
  - incident response

### Acceptance Criteria
- Performance targets met or exception approved with remediation plan.
  - Bulk ingest: `50,000 finding instances <= 120s` (target), stretch target `<= 60s`.
  - Finding transition API/service latency: `p95 <= 200ms`, `p99 <= 500ms` under nominal analyst load.
  - Escalation scan batch (10k open findings): complete `<= 30s` per run.
- SLO evidence must come from target profile (4 vCPU / 8 GB RAM / PostgreSQL 14+) or documented equivalent; local developer machine runs are non-authoritative.
- Reliability tests pass without data corruption or orphan states.
- Operations sign-off completed.

---

## 6. Test and Verification Strategy

1. **Unit Tests** for pure rule logic (risk/SLA/correlation/lifecycle guards).
2. **DB Integration Tests** for repositories and transactional service flows.
3. **Workflow Contract Tests** for escalation and KEV retroactive behavior.
4. **Failure-path Tests** for strict/partial import, retries, and rollback paths.
5. **Performance/Soak Tests** on production-like profile (PostgreSQL-backed).

Verification principle: no completion claim without passing command outputs and stored artifacts.

---

## 7. Rollout Gates (Backend)

All gates must pass before frontend expansion:

1. Phase 1 baseline frozen (committed + tagged).
2. Migration dry-run and rollback drill pass.
3. DB-backed end-to-end backend flow pass:
   - import -> correlate -> risk/sla -> lifecycle -> audit.
4. Audit event completeness validated.
5. Performance benchmark on DB profile meets baseline target.
6. Runbooks and alerting reviewed by operations/security stakeholders.

---

## 8. Risk Register and Mitigations

1. **Data drift during refactor**  
Mitigation: dual-run validation tests and migration rehearsal.

2. **Policy regression under edge dates/timezones**  
Mitigation: timezone matrix tests and canonical calendar utility.

3. **Throughput drop after DB persistence switch**  
Mitigation: indexing plan + batched writes + query profiling.

4. **Connector quality variance**  
Mitigation: strict normalization contract and connector conformance tests.

5. **NocoBase transaction model constraints (nested/cross-scope behavior)**  
Mitigation: BH1 spike + transaction propagation contract tests + prohibition of implicit auto-commit calls inside orchestrated flows.

Decision log note:
- `fqdn` fallback to `row.host` (including IP values) remains accepted in this phase unless downstream reporting requires strict FQDN-only semantics.

---

## 9. Deliverables

- Backend production hardening code changes (repositories/services/workflows).
- Updated and expanded backend test suite (unit + DB integration + perf).
- Updated runbooks with measured evidence.
- Rollout gate report with pass/fail and remediation notes.

---

## 10. Exit Criteria (Production-Ready Backend)

Backend is considered production-ready when:
- Core VM flows are DB-backed and transaction-safe.
- Policy behavior is consistent and documented.
- Audit coverage is complete for critical lifecycle/security events.
- Performance and reliability benchmarks are met under target profile.
- Rollout gates are all green with evidence.

---

## 11. Traceability Matrix (Spec -> Plan)

- Spec `4.1` Persistence/Data Integrity -> Task 0, Sprint BH1 tasks + BH1 acceptance criteria.
- Spec `4.2` Transaction Model -> BH1 spike + BH1 transaction propagation acceptance criteria.
- Spec `4.3` Lifecycle/ACL Correctness -> BH2 ACL recovery correctness tasks + BH2 ACL acceptance criteria.
- Spec `4.4` Risk/SLA/KEV Consistency -> BH2 calendar/KPI/0-row tasks + BH2 acceptance criteria.
- Spec `4.5` Correlation/Source Robustness -> BH2 correlation hardening (including low-confidence `title_hash` fallback queueing) + BH4 source inference hardening.
- Spec `4.6` Audit Completeness -> BH3 audit event set + BH3 acceptance criteria.
- Spec `4.7` Asset Scope Snapshot Integrity -> BH1 asset scope snapshot task + BH3 `asset_scope_changed`.
- Spec `4.8` Connector Runtime Hardening -> BH4 connector runtime tasks + BH4 acceptance criteria.
- Spec `5.1` Performance SLOs -> BH5 numeric targets + SLO evidence requirement.
- Spec `5.2` Reliability -> BH1 rollback safety + BH5 concurrency/reliability tests.
- Spec `5.3` Operability -> BH5 observability + runbook hardening tasks.
- Spec `6` Verification Requirements -> Section 6 Test and Verification Strategy.
- Spec `7` Rollout Gates -> Section 7 Rollout Gates.
