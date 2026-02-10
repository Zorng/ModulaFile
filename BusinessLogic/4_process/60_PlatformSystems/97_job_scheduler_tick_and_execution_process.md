# 97 — Job Scheduler Tick & Execution (March: Billing + Reconciliation)

## Purpose

This process defines the minimum job scheduling and execution pattern required so Modula’s SaaS truth runs without manual intervention:
- renewal at billing anchor,
- grace expiry freeze,
- downgrade effective execution,
- payment reconciliation when webhooks are missed,
- and recovery for "paid but effect not applied".

This is not an analytics pipeline. It is a reliability and enforcement spine.

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/saas_governance/handling_overdue_subscription_and_freeze.md`
- `BusinessLogic/1_stories/saas_governance/upgrading_branch_capabilities_mid_cycle.md`
- `BusinessLogic/1_stories/saas_governance/downgrading_workforce_without_breaking_operations.md`
- `BusinessLogic/1_stories/correcting_mistakes/avoiding_duplicate_actions_due_to_retries.md`

---

## Domains Involved

- Job Scheduler (background execution primitives)
- Subscription & Entitlements (time-based truth + invoices + entitlements)
- Payment (verification via polling/query degradation)
- Webhook Gateway (preferred confirmation signal; may be missed)
- Idempotency (exactly-once effects under retries)
- Audit (canonical state transition evidence, `actor_id = SYSTEM`)

References:
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/job_scheduler_domain.md`
- Edge cases: `BusinessLogic/3_contract/10_edgecases/job_scheduler_edge_case_sweep.md`

---

## When This Process Runs

Triggered by a scheduler tick (implementation-defined) that:
- schedules due jobs, and/or
- executes due jobs.

This process is intentionally implementation-agnostic:
it can be implemented via a cron tick, a managed scheduler, or a worker service.

---

## Canonical Job Kinds (March Baseline)

### JOB-1 — Subscription Renewal Tick
- **Job kind**: `subscription.renewal_tick`
- **Scope**: `tenant_id`
- **Due**: at the tenant billing anchor boundary
- **Owner process**: `BusinessLogic/4_process/60_PlatformSystems/93_subscription_renewal_grace_and_freeze_orchestration.md` (Step 1 + Step 2)
- **Dedupe key**: `sub.renewal:{tenant_id}:{period_end}`

### JOB-2 — Grace Expiry Tick
- **Job kind**: `subscription.grace_expiry_tick`
- **Scope**: `tenant_id`
- **Due**: `past_due_entered_at + 24h`
- **Owner process**: `BusinessLogic/4_process/60_PlatformSystems/93_subscription_renewal_grace_and_freeze_orchestration.md` (Step 3)
- **Dedupe key**: `sub.grace_expiry:{tenant_id}:{past_due_entered_at}`

### JOB-3 — Downgrade Effective Tick
- **Job kind**: `branch.plan_downgrade_effective_tick`
- **Scope**: `tenant_id`, `branch_id`
- **Due**: at or after the downgrade effective date (end-of-cycle)
- **Owner process**: `BusinessLogic/4_process/60_PlatformSystems/92_disable_branch_module_downgrade_pending_process.md` (Step 3)
- **Dedupe key**: `plan.downgrade.apply:{tenant_id}:{branch_id}:{effective_at}`

### JOB-4 — Payment Reconciliation Tick (Invoices)
- **Job kind**: `payment.reconcile_invoices_tick`
- **Scope**: `tenant_id` (or system-wide; implementation choice)
- **Due**: periodic (example: every few minutes) while any invoices are still unpaid/pending
- **Owner processes**:
  - confirmation mechanics: `BusinessLogic/4_process/60_PlatformSystems/95_payment_collection_and_confirmation_orchestration.md` (Step 3, polling path)
  - webhook path: `BusinessLogic/4_process/60_PlatformSystems/96_webhook_ingestion_and_dispatch_process.md` (may be missed)
- **Dedupe key**: `payment.reconcile_invoices_tick:{tick_window_start}`

### JOB-5 — Paid Invoice Effect Recovery Tick
- **Job kind**: `invoice.effect_recovery_tick`
- **Scope**: `tenant_id` (and optional `invoice_id`)
- **Due**: periodic (short interval) to resolve "paid but not unlocked"
- **Owner processes** (apply effect; caller-owned in billing design):
  - first branch activation: `BusinessLogic/4_process/60_PlatformSystems/90_first_branch_activation_orchestration.md` (Step 4)
  - additional branch activation: `BusinessLogic/4_process/60_PlatformSystems/94_additional_branch_activation_orchestration.md` (Step 4)
  - upgrade apply: `BusinessLogic/4_process/60_PlatformSystems/91_enable_branch_module_or_seats_proration_process.md` (Step 4)
  - renewal recovery: `BusinessLogic/4_process/60_PlatformSystems/93_subscription_renewal_grace_and_freeze_orchestration.md` (Step 4)
- **Dedupe key**: `invoice.effect.apply:{invoice_id}`

---

## Execution Rules (Locked)

### Rule A — At-least-once is assumed
Jobs may execute more than once. Handlers must be idempotent.

### Rule B — Single-claim concurrency
Only one worker may execute a job at a time.

### Rule C — Idempotency gate recommendation
Before applying a job’s business effect, use the idempotency gate with:
- `idempotency_key = job_id` (or `dedupe_key`)
- `action_key = job_kind`
- `tenant_id` / `branch_id` as scope

This prevents double effects under retries.

### Rule D — Audit as SYSTEM for time-based truth
When jobs trigger business state transitions (freeze, restore, downgrade complete), audit events must record:
- `actor_id = SYSTEM`
- canonical event names (owned by Subscription/Entitlements)

---

## Orchestration Steps (Generic)

### Step 1 — Schedule due jobs (dedupe-safe)

Scheduler identifies due work and enqueues jobs idempotently using `dedupe_key`.

Examples:
- at billing anchor time, enqueue `subscription.renewal_tick` once per tenant-period
- when entering `PAST_DUE`, enqueue one `subscription.grace_expiry_tick` at +24h
- when downgrade is requested, enqueue one `branch.plan_downgrade_effective_tick` at period end

---

### Step 2 — Claim due jobs (lock)

Workers poll for due jobs and claim atomically:
- select jobs where `due_at <= now` and `status in (SCHEDULED, RETRY_SCHEDULED)`
- claim by setting `status = RUNNING` and `locked_by`, `locked_at`

---

### Step 3 — Execute job handler (idempotent)

Execute handler based on `job_kind` by calling the owner process step(s).

Handler must:
- verify scope exists (tenant/branch/invoice still valid)
- apply business effect idempotently
- return deterministic outcome (success/no-op/retryable failure)

---

### Step 4 — Mark completion or schedule retry

On success:
- set `status = SUCCEEDED` (or keep a "completed_at")

On retryable failure:
- set `status = RETRY_SCHEDULED`
- compute next `due_at` with backoff

On permanent failure:
- set `status = FAILED`
- emit a support signal (operational notification) if it blocks tenant recovery

---

## Failure / Degradation Rules (March)

- Scheduler downtime must not cause irreversible corruption:
  - late jobs run when service recovers.
- Payment reconciliation is required because webhook delivery is not guaranteed.
- Recovery tick must exist to prevent "paid but not unlocked" dead-ends.

