# Job Scheduler Domain Model — Modula POS

## Domain Name
Job Scheduler (Background Execution)

## Domain Type
Platform / Supporting Domain (Time + Reliability Enforcement)

## Domain Group
60_PlatformSystems

## Status
Draft (March baseline: billing enforcement + payment reconciliation)

---

## Purpose

This domain defines how Modula represents **background execution** as first-class system truth:
- time-based transitions (billing anchor renewal, grace expiry freeze, downgrade effective date),
- and reliability recovery (payment reconciliation when webhooks are missed).

It exists because SaaS truth must happen even when:
- no one is logged in,
- the network is unstable,
- or external providers deliver events at-least-once / sometimes never.

This domain is **not analytics**.
Analytics/BI jobs may use the same scheduler later, but are out of scope for March.

---

## Domain Boundary & Ownership

### What This Domain Owns

- The canonical vocabulary for background work:
  - `JobKind`, `JobStatus`, `JobScope`, `dedupe_key`
- The invariants that make background work safe:
  - at-least-once execution assumed
  - idempotent handlers required
  - deterministic locking/claiming to avoid double execution
- Job run evidence (internal observability) separate from tenant audit logs.

### What This Domain Does NOT Own

- Billing state machines and entitlements truth (Subscription & Entitlements owns the business rules).
- Payment proof validation (Payment owns amount/currency/receiver binding).
- Webhook ingestion (Webhook Gateway owns authenticity + normalization).
- Operational workflows (Sale, CashSession, Workforce, Inventory).
- Analytics aggregation and report rollups (explicitly deferred).

The scheduler runs work; it does not define business meaning.

---

## Core Concepts (Ubiquitous Language)

### Job
A durable intent to run a unit of background work.

Key attributes (conceptual):
- `job_id`
- `job_kind` (stable string)
- `dedupe_key` (stable uniqueness key for the intent)
- `scope` (which tenant/branch/invoice is affected)
- `due_at` (UTC timestamp)
- `status` (`SCHEDULED | RUNNING | SUCCEEDED | RETRY_SCHEDULED | FAILED | CANCELED`)
- `attempt_count`
- `locked_at`, `locked_by` (claiming/locking metadata)

### Job Kind
The type of work being run. Examples (March):
- subscription renewal tick
- grace expiry tick
- downgrade apply tick
- payment reconciliation tick
- paid-invoice recovery tick (apply effects after payment confirmed)

### Dedupe Key
A stable string that uniquely identifies one business intent, so jobs can be scheduled repeatedly without duplicates.

Example shapes:
- `sub.renewal:{tenant_id}:{period_end}`
- `sub.grace_expiry:{tenant_id}:{past_due_entered_at}`
- `plan.downgrade.apply:{tenant_id}:{branch_id}:{effective_at}`
- `payment.reconcile.invoice:{invoice_id}`
- `invoice.effect.apply:{invoice_id}`

### At-Least-Once Execution
Workers may run the same job more than once (crash after effect, retries, duplicate claims).

Therefore:
- every job handler must be idempotent,
- and job execution must be safe under duplicates.

---

## Invariants (Non-Negotiable)

- **INV-JOB-1 (Durable intent):** once scheduled, a job must survive process restarts and worker crashes.
- **INV-JOB-2 (At-least-once):** duplicates are assumed; handlers must be idempotent.
- **INV-JOB-3 (Single-claim):** a job must not be processed concurrently by multiple workers.
- **INV-JOB-4 (Deterministic dedupe):** scheduling the same dedupe_key twice must not create two jobs.
- **INV-JOB-5 (UTC time):** job `due_at` is stored and compared in UTC (time zones are presentation, not execution).
- **INV-JOB-6 (No secret leakage):** job payloads/logs must not contain secrets (tokens, raw webhook payloads).

---

## Relationship to Other Domains

- **Subscription & Entitlements**
  - provides time-based triggers (billing anchor, grace window, downgrade effective date).
  - consumes job execution to enforce `PAST_DUE`/`FROZEN` and apply scheduled downgrades.
- **Webhook Gateway + Payment**
  - webhook may confirm payment, but reconciliation jobs are required when webhooks are missed.
  - invoice effect application must remain deterministic and auditable (actor = `SYSTEM`).
- **Idempotency**
  - job handlers should use idempotency gates and/or business unique constraints so duplicates are safe.
- **Audit**
  - business state transitions triggered by jobs must still emit canonical audit events with `actor_id = SYSTEM`.

---

## Out of Scope (For This Domain Doc)

- Analytics/reporting rollups.
- Projection rebuild jobs for Inventory (may use this scheduler later).
- Implementation technologies (cron provider, serverless vs worker process).

---

## Summary

Job Scheduler is a platform domain that makes Modula’s SaaS truth reliable by defining durable jobs,
dedupe semantics, and idempotent background execution for time-based enforcement and payment recovery.

