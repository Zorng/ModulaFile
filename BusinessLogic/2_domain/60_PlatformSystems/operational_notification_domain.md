# Operational Notification Domain Model — Modula POS

## Domain Name
Operational Notification (In-App Signals)

## Domain Type
Platform / Supporting Domain (Coordination Signals)

## Domain Group
60_PlatformSystems

## Status
Draft (Baseline locked for Capstone 1: in-app only; signals not tasks)

---

## Purpose

The Operational Notification domain provides **in-app coordination signals** when an operational state transition creates a human awareness or attention gap.

It exists to help people notice and coordinate around important events (example: a void request pending approval), without becoming part of the business workflow itself.

Operational Notification must remain:
- best-effort (cannot be a correctness dependency)
- idempotent (safe under retries and offline replay)
- access-safe (no cross-tenant or cross-branch leakage)

---

## Domain Boundary & Ownership

### What This Domain Owns

- Notification records (inbox items)
- Recipient state (per-user read/unread)
- Notification typing (notification `type` + `subject`)
- Dedupe/idempotency keys and uniqueness rules
- Read access constraints (who can see which notifications)

### What This Domain Does NOT Own

- The underlying business truth that triggered the notification (Sale, CashSession, Inventory, HR, etc.)
- Approval workflows or enforcement (“must approve within X minutes”)
- Task assignment/claiming (“this manager owns it”)
- Escalation policies (re-notify, paging, external channels)
- External delivery (email/SMS/push) in Capstone 1

Operational Notification is a downstream signal layer, not an orchestration layer.

---

## Core Concepts (Ubiquitous Language)

### OperationalNotification (Entity)

A persisted notification item created by the system with:
- `tenant_id` and (for March baseline) `branch_id`
- `type` (what happened / why it matters)
- `subject_type` + `subject_id` (what it refers to)
- short human-readable message fields (title/body)
- `dedupe_key` (idempotency anchor)

### Recipient (Derived Association)

The set of users who should see a notification is derived from:
- operational responsibility (approver pool, requester, oversight)
- access control rules (must have access to the same tenant/branch context)

Recipients are not “owners of tasks”; they are eligible viewers.

### Read State (Per Recipient)

A recipient may have:
- unread notification
- read notification (timestamped)

Read state is per recipient; it does not change the underlying business state.

### Dedupe Key (Idempotency Anchor)

Each notification type must define a deterministic `dedupe_key` so the same event does not create duplicates under:
- retries
- double event emission
- offline sync replay

---

## Invariants (Non-Negotiable)

- **INV-ON-1 (Signals not tasks):** Notifications inform; they do not assign ownership or guarantee action.
- **INV-ON-2 (Best-effort):** Business truth must remain correct even if notification creation fails.
- **INV-ON-3 (State is authority):** Any UI action taken from a notification must re-check current state; notification freshness is never trusted.
- **INV-ON-4 (Idempotent):** For a given `(tenant_id, dedupe_key)`, at most one notification exists.
- **INV-ON-5 (Access-safe):** A user may only see notifications for tenants/branches they are authorized to access.
- **INV-ON-6 (Branch-scoped baseline):** For March, all operational notifications are branch-scoped and must include `branch_id`.

---

## Trigger Model (Domain-Level)

A valid trigger is:
- a **business state transition** (not a read)
- that creates a waiting/awareness/alert gap
- that does not affect correctness if missed

Typical trigger categories:
- waiting-for-human (strongest): a request pending approval
- outcome feedback: approved/rejected outcome
- operational awareness: informational events (optional)
- exception alert: variance/threshold-based alerts (optional)

Trigger definitions and sequencing belong in Process docs, not here.

---

## Relationship to Other Domains

- **Sale/Order + POSOperation processes**: emit notifications for void request pending and outcomes (best-effort).
- **Cash Session**: may emit informational/alert notifications on close and variance (best-effort).
- **Access Control**: defines who is eligible to receive a notification in a given context.
- **Offline Sync**: retries/replays events; Operational Notification must be idempotent under replay.
- **Audit**: business truth changes are audited at the source; Operational Notification may also log creation/view events (optional).

---

## Out of Scope (For This Domain Doc)

- Cross-channel delivery (push/email/SMS)
- Escalation/assignment/task management
- Notification preferences per user
- HR notifications (future extension; baseline triggers are POSOperation)

---

## Summary

Operational Notification is a platform domain that provides **in-app signals** for operational coordination, while remaining:
- non-authoritative for business truth,
- replay-safe,
- and access-safe.

