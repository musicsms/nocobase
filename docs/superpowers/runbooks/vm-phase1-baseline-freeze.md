# VM Phase 1 Baseline Freeze

**Date:** 2026-05-09  
**Purpose:** Freeze Phase 1 implementation baseline before Phase 2 backend production hardening.

## Baseline Commit
- Baseline anchor commit: `5a2e8cfeab`
- Freeze tag: `vm-phase1-baseline`

## Verification Evidence
Executed on 2026-05-09:
- `yarn test packages/plugins/@nocobase/plugin-vulnerability-management/src/server/__tests__/plugin-bootstrap.test.ts` (pass)
- `yarn test packages/plugins/@nocobase/plugin-vulnerability-management/src/server/__tests__/ingestion-partial-and-asset.test.ts` (pass)
- `yarn test packages/plugins/@nocobase/plugin-vulnerability-management/src/server/__tests__/risk-sla.test.ts` (pass)
- `yarn test packages/plugins/@nocobase/plugin-vulnerability-management/src/server/__tests__/lifecycle-guards.test.ts` (pass)
- `yarn test packages/plugins/@nocobase/plugin-vulnerability-management/src/server/__tests__/correlation.test.ts` (pass)
- `yarn test packages/plugins/@nocobase/plugin-vulnerability-management/src/server/__tests__/sla-escalation.test.ts` (pass)
- `yarn test packages/plugins/@nocobase/plugin-vulnerability-management/src/server/__tests__/vm-phase1-e2e.test.ts` (pass)
- `yarn test packages/plugins/@nocobase/plugin-vulnerability-management/src/server/__tests__/kev-retroactive.test.ts` (pass)
- `yarn test packages/plugins/@nocobase/plugin-vulnerability-management/src/server/__tests__/connector-framework.test.ts` (pass)
- `yarn test packages/plugins/@nocobase/plugin-vulnerability-management/src/server/__tests__/ingestion-perf.test.ts` (pass)

## Carry-over Debt to Phase 2
- openCount KPI currently counts `false_positive` as open.
- Correlation key strategy needs stronger cross-version resilience.
- Recovery ACL edge `accepted_risk -> triaged` role behavior hardening.
- Audit writer pattern must be DB-injected and complete event set.
- Perf evidence is currently in-memory and must be re-benchmarked on DB profile.
- `0-row` ingestion status semantics needs explicit final behavior.
- E2E still does not execute full import chain in one scenario.
- Source inference is still fragile when filename metadata is absent.
- `fqdn` fallback may contain IP value and remains accepted limitation for now.
