# Edge Case Contract — Job Scheduler (Background Jobs)

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: background job execution safety (billing ticks, downgrade enforcement, payment reconciliation)
- **Primary Audience**: Backend, QA
- **Owner(s)**: PlatformSystems (Job Scheduler), Subscription/Entitlements, Payment, Webhook Gateway, Idempotency, Audit
- **Last Updated**: 2026-02-11
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Domain**:
  - `BusinessLogic/2_domain/60_PlatformSystems/job_scheduler_domain.md`
- **Related Processes**:
  - `BusinessLogic/4_process/60_PlatformSystems/97_job_scheduler_tick_and_execution_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/93_subscription_renewal_grace_and_freeze_orchestration.md`
  - `BusinessLogic/4_process/60_PlatformSystems/92_disable_branch_module_downgrade_pending_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/95_payment_collection_and_confirmation_orchestration.md`
  - `BusinessLogic/4_process/60_PlatformSystems/96_webhook_ingestion_and_dispatch_process.md`

---

## Purpose

Lock the minimum background job behaviors needed for March so that:
- subscription enforcement happens on time,
- paid invoices do not get stuck in "paid but not unlocked",
- and duplicate job execution does not create duplicate business effects.

---

## Definitions / Legend

- **Job**: durable background intent with `job_kind`, `dedupe_key`, `due_at`, and `status`.
- **At-least-once**: a job may execute more than once (retries, worker crash, duplicate delivery).
- **Handler idempotency**: running the same handler multiple times yields one business outcome.

---

## Edge Case Catalog (Locked)

### EC-JOB-01 — Duplicate job execution (at-least-once)
- **Scenario**: A worker crashes after applying a business effect but before marking job success; job executes again.
- **Trigger**: worker crash, timeout, retry policy.
- **Expected Behavior**:
  - Handler must be idempotent: second execution is a no-op and returns the same outcome.
  - No duplicate invoices, branches, entitlement changes, or state transitions are created.
- **Owner**: Job Scheduler + Idempotency + downstream domains
- **March**: Yes

### EC-JOB-02 — Two workers attempt to claim the same job
- **Scenario**: Multiple workers poll due jobs concurrently.
- **Trigger**: concurrency race.
- **Expected Behavior**:
  - Only one worker claims the job (DB lock/atomic update).
  - Other workers must not run the same job concurrently.
- **Owner**: Job Scheduler
- **March**: Yes

### EC-JOB-03 — Job is delayed (runs late)
- **Scenario**: Scheduler downtime causes the renewal/grace job to run late.
- **Trigger**: deployment outage, provider downtime.
- **Expected Behavior**:
  - Job runs "as soon as possible" after recovery.
  - Business rules still apply deterministically based on current time:
    - renewal invoice issuance happens once,
    - grace expiry freeze happens if overdue past 24h.
- **Owner**: Job Scheduler + Subscription/Entitlements
- **March**: Yes

### EC-JOB-04 — Renewal invoice issued twice
- **Scenario**: Renewal tick runs twice and issues two renewal invoices for the same period.
- **Trigger**: missing dedupe, handler not idempotent.
- **Expected Behavior**:
  - Must never happen: invoice issuance is deduped by a stable period key.
  - Second attempt returns existing invoice.
- **Owner**: Subscription/Entitlements + Job Scheduler
- **March**: Yes

### EC-JOB-05 — Payment confirmed but entitlements/branch not activated (stuck paid)
- **Scenario**: Invoice is marked `PAID` but the corresponding effect did not complete (branch provision, upgrade apply, state restore).
- **Trigger**: crash between "mark paid" and "apply effect", webhook arrived during outage.
- **Expected Behavior**:
  - A recovery job must detect and complete the missing effect idempotently.
  - Tenant should not be forced into manual support for this class of failure in March.
- **Owner**: Job Scheduler + Subscription/Entitlements
- **March**: Yes

### EC-JOB-06 — Webhook missed; invoice remains unpaid in-app
- **Scenario**: Tenant pays, but webhook delivery fails; invoice remains pending.
- **Trigger**: provider outage, webhook endpoint downtime.
- **Expected Behavior**:
  - Payment reconciliation job polls and confirms payment deterministically.
  - Once confirmed, invoice is marked paid and effect recovery applies unlock/provision.
- **Owner**: Job Scheduler + Payment + Webhook Gateway (degradation path)
- **March**: Yes

### EC-JOB-07 — Downgrade pending cannot complete due to active staff
- **Scenario**: Workforce downgrade is effective-date reached, but safe boundary is not satisfied.
- **Trigger**: active attendance or open staff cash session.
- **Expected Behavior**:
  - Downgrade remains in `DOWNGRADE_PENDING`.
  - Job retries later until the safe boundary holds.
  - No mid-operation bricking occurs.
- **Owner**: Subscription/Entitlements + Workforce + Job Scheduler
- **March**: Yes

### EC-JOB-08 — Clock/time zone ambiguity
- **Scenario**: Tenant is in a different time zone; renewal anchor is misinterpreted.
- **Trigger**: using client time, mixing local and UTC, DST bugs.
- **Expected Behavior**:
  - Execution uses server UTC times only.
  - Any "local time" presentation is UI-only and must not affect execution truth.
- **Owner**: Job Scheduler + Subscription/Entitlements
- **March**: Yes

---

## Summary

For March, background jobs must be:
- durable,
- safe under duplicates,
- concurrency-safe under multiple workers,
- able to reconcile missed webhooks,
- and able to complete "paid but not unlocked" states automatically.

