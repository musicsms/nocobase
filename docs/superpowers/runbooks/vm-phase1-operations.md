# VM Phase 1 Operations Runbook

## Purpose
Operate ingestion, triage, remediation, and escalation flows for VM Phase 1.

## Daily Checks
- Verify ingestion jobs completed and inspect partial failures.
- Download reject CSV and route parsing/data errors to source teams.
- Review near-due and overdue escalation events.
- Monitor queue load for triage and remediation owners.

## Weekly Checks
- Review KPI dashboard: open/closed counts, time-to-triage, MTTR.
- Sample audit logs for lifecycle transitions and role compliance.
- Re-run correlation dry-run and validate candidate quality before apply.

## Incident Handling
- If ingestion backlog exceeds target, pause non-critical imports.
- If SLA escalation spikes, notify vm_admin and service owners.
- If bad data introduced, use rollback runbook immediately.

## Verification
- Run `yarn test` for plugin tests before release changes.
- Capture elapsed time from ingestion perf harness for trend tracking.
- Note: current perf harness is in-memory (`VmFindingRecord[]`) and is not a DB throughput benchmark.
- Note: current E2E scenario validates lifecycle/KPI/audit path after finding creation; parse/normalize/ingest path is covered by Task 3-4 tests.

## KPI Semantics (Phase 1)
- `closedCount`: only findings in state `closed`.
- `openCount`: `total - closedCount` (currently includes `false_positive` and `accepted_risk`).
- Before production reporting sign-off, confirm whether `false_positive` and `accepted_risk` should be reclassified as closed-like.

## Open Items For Phase 2
- 🟡 `openCount` KPI currently counts `false_positive` as open; needs finalized closed-like state policy.
- 🟡 Correlation composite key includes `title`; may miss merges when scanner title changes across versions.
- 🟡 `vm_triage_queue` path does not restore `accepted_risk -> triaged` behavior.
- 🟡 Shared audit logger is singleton-style in test architecture; production path should inject DB-backed audit writer.
- 🟡 `ingestion-perf` harness is in-memory and does not measure real DB throughput/latency.
- 🟡 0-row ingestion currently maps to `jobStatus='failed'`, which may be misleading vs empty-input semantics.
- 🟡 E2E test does not execute full import chain (parse -> normalize -> ingest), only post-create lifecycle path.
- 🟡 Source inference relies on filename patterns and is fragile with UUID/object-store filenames.
- 🟡 KEV due-date calculation currently uses calendar-day math while near-due/overdue checks use business-day logic.
- 🟢 `fqdn` fallback to `row.host` can produce IP values; verify downstream reporting expectations.
