# VM Phase 1 Rollback Runbook

## Trigger Conditions
- Correlation apply merges incorrect findings.
- Escalation workflow emits wrong recipients or event type.
- KPI or lifecycle regressions after deployment.

## Rollback Procedure
1. Disable feature flag: set `VM_PHASE1_ENABLED=false`.
2. Stop scheduled imports and escalation workflow executions.
3. Revert plugin commit for the release window.
4. Re-run migration safety checks before any re-enable.

## Data Recovery
- Restore affected findings from DB backup snapshot.
- Clear/repair `merged_into_id` where correlation apply was incorrect.
- Recompute KPI snapshots after data restore.

## Exit Criteria
- Critical workflows restored and validated by test suite.
- vm_admin signs off on escalation and lifecycle behavior.
- Feature flag re-enabled only after verification evidence is archived.
