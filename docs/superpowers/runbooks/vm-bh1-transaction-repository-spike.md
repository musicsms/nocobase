# VM BH1 Spike: NocoBase Transaction and Repository Contract

**Date:** 2026-05-10  
**Scope:** Backend hardening BH1 design spike

## Purpose
Define a concrete transaction/repository contract for VM services on NocoBase to avoid partial writes and hidden auto-commit behavior.

## Contract
1. All persistent writes go through collection repositories from `app.db.getRepository('<collection>')`.
2. Multi-collection flows MUST execute inside one `app.db.sequelize.transaction(...)`.
3. Transaction context MUST be passed to every repository call in that flow.
4. Repository methods are thin wrappers and must accept `VmTransactionContext`.
5. Orchestration services own transaction boundaries; repositories must not open nested transactions by default.

### Orchestration usage pattern (required)

```ts
await withVmTransaction(app, async (ctx) => {
  const job = await jobRepo.create(jobInput, ctx);
  const { record: finding } = await findingRepo.upsertByFingerprint(fingerprint, findingInput, ctx);
  await instanceRepo.create({ ...instanceInput, finding_id: finding.id, ingestion_job_id: job.id }, ctx);
  await rejectRepo.bulkCreate(rejectRows, ctx);
});
```

## Target Collections (BH1)
- `vm_findings`
- `vm_finding_instances`
- `vm_ingestion_jobs`
- `vm_ingestion_rejects`
- `vm_exceptions`
- `vm_remediation_tasks`
- `vm_activity_logs`

## Initial Skeleton Added
- `src/server/repositories/transaction-context.ts`
- `src/server/repositories/finding-repository.ts`
- `src/server/repositories/finding-instance-repository.ts`
- `src/server/repositories/ingestion-job-repository.ts`
- `src/server/repositories/ingestion-reject-repository.ts`
- `src/server/repositories/exception-repository.ts`
- `src/server/repositories/remediation-task-repository.ts`
- `src/server/repositories/activity-log-repository.ts`
- `src/server/repositories/index.ts`

## Next BH1 Implementation Steps
1. Refactor `src/server/services/ingestion-service.ts` and `src/server/resources/ingestion-resource.ts`:
   - create/update `vm_ingestion_jobs`
   - write `vm_ingestion_rejects`
   - create/update `vm_findings` + `vm_finding_instances`
   - all in one transaction.
2. Refactor `src/server/services/fingerprint-service.ts`:
   - delegate upsert to `VmFindingRepository.upsertByFingerprint(...)`
   - persist pre-update `asset_scope` snapshot before overwrite.
3. Add DB integration tests under `src/server/__tests__/` proving rollback safety across findings/instances/jobs/rejects writes.

## Current Test Scope Note
- Current `*-db-service.test.ts` files are repository-contract tests with mocked app/repository surfaces.
- They validate transaction-context propagation and orchestration behavior, but are **not** a substitute for real DB integration tests.
- Real Sequelize/DB-backed integration tests remain mandatory before BH1 is marked complete.
