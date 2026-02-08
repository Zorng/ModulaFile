# Audit Logging Module — Core Module (Capstone I)

**Version:** 1.2  
**Status:** Patched (Attendance events aligned; frozen-branch denials added)  
**Module Type:** Core Module  
**Depends on:** Authentication & Authorization (Core), Tenant & Branch Context (Core), Sync & Offline Support (Core)  
**Used by:** Sale, Cash Session & Reconciliation, Attendance, Menu, Inventory, Discount, Policy & Configuration, Tenant/Branch (Core)

---

## 1. Purpose

The Audit Logging module provides a **tamper-resistant, read-only record of business-critical actions** performed within Modula.

Its purpose is to:
- ensure accountability of users and roles,
- support investigation of disputes, errors, or suspicious behavior,
- provide traceability across sales, cash handling, attendance, inventory, and configuration changes,
- maintain trust in Modula as a management-grade POS.

Audit logs are **not analytics** and **not debugging logs**.  
They exist to answer: **who did what, when, where, and with what outcome**.

---

## 2. Scope & Boundaries

### Includes
- Logging of **successful and rejected/failed business actions**
- Logging across **all modules** that mutate business state
- Logging of **blocked actions** (policy/permission/branch freeze)
- Support for **offline-generated events** with idempotent ingestion
- Branch-scoped and tenant-scoped events
- Admin-only read access to audit records

### Excludes
- Real-time alerts
- Analytics, metrics, or dashboards
- Editable or deletable logs
- System performance or error stack traces
- External integrations (SIEM, exports) beyond read-only UI

---

## 3. Visibility & Access Control

- **Admin**:
  - Can view all audit logs across all branches
- **Manager / Cashier**:
  - No access to audit logs
- **System**:
  - May generate audit events (e.g., sync recovery)

Audit logs are **read-only** for tenant users and cannot be modified, deleted, or backdated.

---

## 4. Core Concepts

### 4.1 Audit Event
An audit event represents a **single business action attempt** that resulted in either:
- a successful state change, or
- a rejected/failed outcome due to validation, policy, or permission rules.

### 4.2 Outcome
Each audit event records an outcome:
- `SUCCESS`
- `REJECTED`
- `FAILED`

### 4.3 Actor
An audit event may be triggered by:
- a human user (Admin / Manager / Cashier), or
- the system (e.g., background sync, recovery)

### 4.4 Denial Reasons (standardized)
When outcome is `REJECTED`, the audit event should include a **denial reason**:
- `PERMISSION_DENIED`
- `POLICY_BLOCKED`
- `VALIDATION_FAILED`
- `BRANCH_FROZEN`
- `TENANT_FROZEN` (future-safe)
- `DEPENDENCY_MISSING` (e.g., required cash session missing)

> Note: `BRANCH_FROZEN` is used when an action is blocked because the branch status is frozen/disabled. This enables later investigation and reporting of operational disruptions.

---

## 5. Use Cases

### UC-1 — Record Business Action (Success)
**Actors:** System

**Preconditions**
- A business action completes successfully in any module

**Main Flow**
1. Module emits an audit event describing the action.
2. Event includes actor, module, entity, and outcome = SUCCESS.
3. Event is persisted.

**Postconditions**
- Audit record exists and is immutable.

---

### UC-2 — Record Rejected or Failed Action
**Actors:** System

**Preconditions**
- A business action is blocked by policy, permission, validation, or frozen branch

**Main Flow**
1. Module emits an audit event with outcome = REJECTED or FAILED.
2. Event includes rejection reason (e.g., BRANCH_FROZEN) and attempted action/entity.
3. Event is persisted.

**Postconditions**
- Audit record exists describing the denied attempt.

---

### UC-3 — Offline Action Logging
**Actors:** System

**Preconditions**
- Client operates offline and performs state-changing actions

**Main Flow**
1. Client generates audit events with a client-generated event ID.
2. Events are queued locally.
3. Upon reconnect, events are synced.
4. Backend ingests events idempotently.

**Postconditions**
- No duplicate audit entries
- Original event timestamps preserved

---

### UC-4 — View Audit Logs (Admin)
**Actors:** Admin

**Preconditions**
- Admin authenticated
- Tenant context active

**Main Flow**
1. Admin opens Audit Log screen.
2. Admin filters by date, branch, module, actor, or outcome.
3. Admin selects a log entry to view details.

**Postconditions**
- No data mutation occurs.

---

## 6. Logged Event Categories (Capstone I)

### Branch Lifecycle (NEW)
- BRANCH_PROVISIONED
- BRANCH_UPDATED
- BRANCH_FROZEN
- BRANCH_UNFROZEN

### Authentication
- LOGIN_SUCCESS
- LOGIN_FAILED

### Sale & Orders
- SALE_FINALIZED
- VOID_REQUESTED
- VOID_APPROVED
- VOID_REJECTED
- ORDER_STATUS_CHANGED
- ORDER_CANCELLED_BY_VOID

### Cash Session
- CASH_SESSION_OPENED
- CASH_SESSION_CLOSED
- CASH_SESSION_FORCE_CLOSED
- CASH_PAID_IN
- CASH_PAID_OUT_REQUESTED
- CASH_PAID_OUT_APPROVED
- CASH_PAID_OUT_REJECTED
- CASH_ADJUSTMENT_POSTED

### Attendance
- CHECK_IN_CREATED
- CHECK_OUT_CREATED
- CHECK_OUT_FORCED (optional; otherwise enrich CHECK_OUT_CREATED with forced metadata)

### Policy & Configuration
- POLICY_UPDATED

### Menu
- MENU_ITEM_CREATED
- MENU_ITEM_UPDATED
- MENU_ITEM_ARCHIVED
- MODIFIER_GROUP_CREATED
- MODIFIER_GROUP_UPDATED

### Inventory
- STOCK_ITEM_CREATED
- STOCK_ITEM_UPDATED
- STOCK_ITEM_ARCHIVED
- RESTOCK_BATCH_CREATED
- INVENTORY_JOURNAL_POSTED

### Discount
- DISCOUNT_CREATED
- DISCOUNT_UPDATED
- DISCOUNT_DISABLED
- DISCOUNT_ASSIGNMENT_CHANGED

### Cross-cutting Denials (NEW, optional but recommended)
These are not “features”; they are outcomes that should be logged when relevant:
- ACTION_REJECTED_BRANCH_FROZEN
  - Use when a user attempts a state-changing action on a frozen branch (sale, restock, cash open, check-in, etc.)
  - Outcome = REJECTED, reason = BRANCH_FROZEN

> Implementation note: You may represent this either as (a) a dedicated event name above, or (b) the original event name with outcome=REJECTED and reason=BRANCH_FROZEN. Pick one consistent approach across modules.

---

## 7. Requirements

- R1: Every successful state-changing business action must emit an audit event.
- R2: Rejected or failed attempts must also be logged.
- R3: Each audit event must include tenant context.
- R4: Branch context must be recorded when applicable.
- R5: Actor identity and role must be captured when available.
- R6: Audit logs must be immutable once stored.
- R7: Offline-generated events must be deduplicated safely.
- R8: Audit logs must not contain sensitive credentials or secrets.
- R9: Branch lifecycle actions must emit audit events (provision, update, freeze, unfreeze).
- R10: If a branch is frozen, any blocked state-changing attempt should be auditable (at minimum: outcome=REJECTED + reason=BRANCH_FROZEN).

---

## 8. Acceptance Criteria

- AC-1: Admin can view audit logs across all branches.
- AC-2: Managers and Cashiers cannot access audit logs.
- AC-3: Failed or rejected actions appear in the audit log with reasons.
- AC-4: Offline actions appear exactly once after sync.
- AC-5: Audit entries cannot be edited or deleted.
- AC-6: Each audit entry clearly shows actor, action, entity, and outcome.
- AC-7: Branch lifecycle actions are logged (BRANCH_PROVISIONED/UPDATED/FROZEN/UNFROZEN).
- AC-8: Actions blocked due to frozen branch are recorded as REJECTED with reason BRANCH_FROZEN (either as a dedicated denial event or as the original action event).

---

## 9. Retention & Compliance

- Audit logs are retained for the duration of the tenant’s subscription (Capstone I).
- Retention policies may become configurable in future phases.
- Audit data is isolated per tenant and never shared across tenants.

---

## 10. Out of Scope (Future Work)

- Alerting or notifications based on audit events
- Export to external compliance systems
- Analytics dashboards
- Automated anomaly detection

---

# End of Document
