# Job Scheduler Module — Platform Module (Background Jobs)

**Version:** 1.0  
**Status:** Draft (March baseline: billing enforcement + payment reconciliation; analytics deferred)  
**Module Type:** Platform / Supporting Module  
**Depends on:** Idempotency (exactly-once effects), Subscription & Entitlements (business truth), Payment (polling verification), Webhook Gateway (preferred confirmations), Audit (SYSTEM evidence)  
**Used by:** Subscription & Entitlements processes; future platform tasks

---

## 1. Purpose

This module defines how Modula runs **background jobs** so time-based and reliability-based truths happen without manual intervention.

It exists to support March requirements:
- renewal at billing anchor
- grace expiry freeze
- downgrade effective execution
- invoice payment reconciliation when webhook is missed
- recovery for "paid but effect not applied"

This module is not an analytics engine.

Authoritative references:
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/job_scheduler_domain.md`
- Contract: `BusinessLogic/3_contract/10_edgecases/job_scheduler_edge_case_sweep.md`
- Process: `BusinessLogic/4_process/60_PlatformSystems/97_job_scheduler_tick_and_execution_process.md`

---

## 2. Scope (March Baseline)

Included:
- durable job scheduling using `dedupe_key`
- due job execution with single-claim locking
- at-least-once execution semantics (handlers must be idempotent)
- minimal retry strategy with backoff
- job kinds needed for subscription/billing enforcement and recovery

Excluded (explicit):
- analytics rollups and report pre-aggregation
- inventory projection rebuild jobs (future)
- tenant-facing job management UI

---

## 3. Job Model (Conceptual)

### Job fields (minimum)
- `job_id`
- `job_kind`
- `dedupe_key`
- `due_at` (UTC)
- `status` (`SCHEDULED | RUNNING | SUCCEEDED | RETRY_SCHEDULED | FAILED | CANCELED`)
- `attempt_count`
- `locked_by`, `locked_at`
- `scope_refs` (ids only: `tenant_id`, optional `branch_id`, optional `invoice_id`)

### Dedupe rule (locked)
Scheduling the same `dedupe_key` twice must return the existing job (no duplicates).

---

## 4. Job Kinds (March)

Canonical kinds are defined by the process doc:
- `subscription.renewal_tick`
- `subscription.grace_expiry_tick`
- `branch.plan_downgrade_effective_tick`
- `payment.reconcile_invoices_tick`
- `invoice.effect_recovery_tick`

---

## 5. Interfaces (Conceptual)

- `EnqueueJob(job_kind, dedupe_key, due_at, scope_refs) -> job_id`
- `RunDueJobs(now_utc, limit) -> run_summary`
- `CancelJob(dedupe_key) -> outcome` (optional)

The module does not define HTTP endpoints; it defines behaviors the implementation must support.

---

## 6. Idempotency & Safety

### 6.1 Job execution must be idempotent
Jobs may execute more than once. Handlers must be safe under duplicates.

Recommended pattern:
- wrap each job handler with Idempotency Gate using `job_id` (or `dedupe_key`) as `idempotency_key`
- downstream processes must also be idempotent for their own invariants (invoice issuance, branch provision, entitlement activation).

### 6.2 Audit evidence for background-driven truth
When a job triggers business truth transitions:
- record canonical audit events with `actor_id = SYSTEM`
- keep event names owned by the domain processes (Subscription/Entitlements)

---

## 7. Acceptance Criteria (March)

- AC-1: Renewal invoice is issued exactly once per tenant-period (no duplicates under retries).
- AC-2: `PAST_DUE` grace expiry reliably enters `FROZEN` if unpaid after 24h.
- AC-3: Paid invoices never remain stuck without their effect applied (branch provision, upgrade apply, restore active).
- AC-4: Downgrade effective jobs never brick operations mid-session; Workforce remains pending until safe boundary.
- AC-5: Missed webhooks are reconciled by polling and unlock/provision still completes.

