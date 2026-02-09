# Audit Logging Domain Model — Modula POS

## Domain Name
Audit Logging (Business Activity Log)

## Domain Type
Platform / Supporting Domain (Accountability)

## Domain Group
60_PlatformSystems

## Status
Draft (Baseline for Capstone 1: tenant audit log for business-critical actions)

---

## Purpose

The Audit Logging domain provides a **tamper-resistant, immutable record** of business-critical actions performed in Modula.

It exists to support:
- accountability (“who did what”)
- dispute investigation (“what happened”)
- traceability across modules (sales, cash handling, inventory, HR, configuration)

Audit logs are **not analytics** and not debugging logs.
They exist to preserve operational evidence.

---

## Domain Boundary & Ownership

### What This Domain Owns

- The audit event record model (schema + invariants)
- Append-only storage rules (immutability)
- Idempotent ingestion rules (safe under retries/offline replay)
- Read/query interface rules (filtering, pagination)
- Access-safety constraints (no cross-tenant leakage)

### What This Domain Does NOT Own

- Business truth and state machines (Sale, CashSession, Inventory, HR, etc.)
- Reporting/analytics aggregations
- Operational alerts/coordination (Operational Notification)
- Authorization policy (Access Control decides who may view; Audit enforces by checking)
- Offline sync transport mechanics (owned by Offline Sync)

Audit stores evidence; it does not decide business outcomes.

---

## Core Concepts (Ubiquitous Language)

### AuditEvent (Entity)

A single record representing an **attempted business action** with:
- `event_type` (stable string, module-owned)
- `tenant_id` (required for in-tenant business events)
- `branch_id` (required when the action is branch-scoped)
- `actor_id` (who initiated it; may be system)
- `occurred_at` (when the action happened / was attempted)
- `recorded_at` (when the backend persisted it)
- `outcome` (`SUCCESS | REJECTED | FAILED`)
- optional `reason_code` for non-success outcomes
- `subject` and `entity` references (what it affected)
- `metadata` (small, non-sensitive context for investigation)

### Outcome

- `SUCCESS`: business state changed successfully (or an observational event was recorded)
- `REJECTED`: denied before state mutation (authorization, branch frozen, validation, dependency missing)
- `FAILED`: attempted but failed due to system failure (timeouts, exceptions)

### Idempotency Key

A stable key that prevents duplicates under:
- retries
- double submits
- offline sync replay

For Capstone 1, the minimum is:
- a client operation ID (offline-safe), or
- a deterministic server key per action/entity when applicable.

### Event Type Ownership (Anti-Drift Rule)

Audit Logging does **not** define every event type centrally.

Instead:
- each module’s ModSpec defines the event types it emits
- Audit enforces the schema/invariants and stores them consistently

This prevents “central catalog drift” where an audit spec becomes stale while modules evolve.

Naming convention (recommended for Capstone 1):
- `UPPER_SNAKE_CASE` event type strings (example: `SALE_FINALIZED`, `CASH_SESSION_OPENED`)

---

## Invariants (Non-Negotiable)

- **INV-AUD-1 (Immutable):** Audit events are append-only; no updates or deletes in Capstone 1.
- **INV-AUD-2 (Tenant isolation):** Every in-tenant business audit event must include `tenant_id`.
- **INV-AUD-3 (Branch explicitness):** If the audited action is branch-scoped, `branch_id` must be present.
- **INV-AUD-4 (Atomicity for state changes):** For state-changing operations, the audit record must be persisted atomically with the state change.
- **INV-AUD-5 (Idempotent ingestion):** Duplicate submissions of the same event must not create duplicate audit entries.
- **INV-AUD-6 (No secrets):** Audit metadata must not contain credentials or sensitive secrets.
- **INV-AUD-7 (Outcome clarity):** Non-success outcomes must include a reason code that is stable and explainable.

### Reason Codes (Alignment Rule)

When a rejection is caused by Access Control, record the Access Control `reason_code` as the audit `reason_code`.

For non-authorization rejections, allowed baseline reason codes include:
- `VALIDATION_FAILED`
- `DEPENDENCY_MISSING`
- `BUSINESS_RULE_BLOCKED`

Additional codes may exist, but must remain stable once introduced.

---

## Visibility & Access (March Baseline)

- Audit log viewing is restricted to **Owner/Admin** users.
- Managers/Cashiers have no access to audit logs in Capstone 1.

Access control enforcement uses the Access Control domain.
`audit.view` is treated as a **TENANT-scoped** action for March (governance visibility).

---

## Relationship to Other Domains

- **Access Control**: decides if `audit.view` is allowed; provides reason codes for denied actions.
- **Offline Sync**: replays operations; Audit must be idempotent under replay.
- **All business domains**: emit audit events when they mutate state.
- **Operational Notification**: may reference audited actions but is not a substitute for audit evidence.

---

## Out of Scope (For This Domain Doc)

- Analytics dashboards and derived KPIs
- Alerting/notification policies driven by audit events
- Export/forwarding to external systems
- Configurable retention policies

---

## Summary

Audit Logging is a platform domain that stores immutable evidence of business-critical actions.
It is intentionally kept separate from reporting and notifications, while remaining replay-safe and access-safe.
