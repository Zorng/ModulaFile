# Cash Session Domain Model — Modula POS

**Domain Name:** Cash Session  
**Domain Type:** Core Business Domain  
**Status:** Defined (Aligned with Capstone 2 domain modeling)  
**Scope:** Food & Beverage (single-register first)  
**Depends On:** Tenant, Branch, Order, Audit Logging  
**Last Updated:** 2026-01

---

## 1. Domain Purpose & Responsibility

The Cash Session domain defines the **cash accountability window** for a branch.  
Its purpose is to ensure that all cash movements are traceable, reconcilable, and auditable within a clearly bounded operational period.

The Cash Session domain is responsible for:
- defining when cash handling begins and ends,
- recording all cash inflows and outflows immutably,
- computing reconciliation results (expected vs counted),
- enforcing integrity constraints for cash-based sales and cash refunds (including approved voids).

The Cash Session domain exists to protect **financial accountability**, not to manage sales logic, attendance, or inventory behavior.

---

## 2. Domain Boundary & Ownership

### 2.1 What This Domain Owns
- The lifecycle of a cash accountability window for a branch.
- The append-only ledger of cash movements.
- Reconciliation results and discrepancy tracking.
- Enforcement of “cash session must be OPEN” for sale finalization (Capstone I product rule) and for any cash-out refund movement.

### 2.2 What This Domain Does NOT Own
- Pricing, discounts, VAT, FX, or rounding (Order domain).
- Order composition or fulfillment state (Order domain).
- Inventory stock rules or reservations (Inventory domain).
- Staff shift scheduling or attendance evaluation (Attendance domain).
- Tenant or branch provisioning (Tenant / Branch domain).
- Offline delivery guarantees (Offline Sync domain).

---

## 3. Core Concepts (Ubiquitous Language)

| Term | Meaning |
|----|----|
| Cash Session | A bounded period during which cash handling is accountable |
| Opening Float | Initial cash amount at session start |
| Cash Movement | Immutable record of cash delta |
| Reconciliation | Comparison of expected vs counted cash |
| Force Close | Administrative termination of a session without full reconciliation |
| Drawer | Conceptual cash container represented by a session |

---

## 4. Aggregate & Entities

### Aggregate Root: **CashSession**

The CashSession aggregate defines the **consistency boundary** for all cash movements.

#### CashSession Entity
- cash_session_id
- tenant_id
- branch_id
- opened_by_auth_account_id
- opened_at
- status (OPEN | CLOSED | FORCE_CLOSED)
- opening_float (Money snapshot)
- closed_at (nullable)
- close_note (optional)

Note: `*_auth_account_id` refers to the global actor identity key from the Authentication domain (not a phone number identifier).

#### Child Entity: **CashMovement**
- cash_movement_id
- cash_session_id
- movement_type
- amount (Money)
- reason
- source_reference (sale_id / void_id / manual)
- recorded_by_auth_account_id
- created_at
- idempotency_key (required for replay-safe movements)

CashMovement is **append-only** and immutable.

---

## 5. Lifecycle & State Model

### 5.1 CashSession States

| State | Meaning |
|----|----|
| OPEN | Active accountability window |
| CLOSED | Normally closed with reconciliation |
| FORCE_CLOSED | Administratively closed without full reconciliation |

### 5.2 Allowed Transitions

| From | To | Trigger |
|----|----|----|
| none | OPEN | OpenCashSession |
| OPEN | CLOSED | CloseCashSession |
| OPEN | FORCE_CLOSED | ForceCloseCashSession |

### 5.3 Forbidden Transitions
- CLOSED → OPEN
- FORCE_CLOSED → CLOSED
- Any transition that deletes or mutates existing CashMovements

---

## 6. Commands (Intent-Based)

### 6.1 Cash Session Commands

| Command | Intent |
|----|----|
| OpenCashSession | Begin a new accountability window |
| CloseCashSession | End session with reconciliation |
| ForceCloseCashSession | End session administratively |

### 6.2 Cash Movement Commands

| Command | Intent |
|----|----|
| RecordSaleCashIn | Cash received from finalized sale |
| RecordRefundCashOut | Cash refunded for an approved void/refund (append-only) |
| RecordPaidIn | Manual cash added |
| RecordPaidOut | Manual cash removed |
| RecordAdjustment | Explicit correction (restricted) |

All movement commands **append new records** only.

Terminology note: “Reverse cash effects” for a voided cash sale means appending a refund cash-out movement (e.g., `REFUND_CASH`) that compensates the original sale cash-in movement. Nothing in the cash ledger is edited or deleted.

---

## 7. Business Invariants (Hard Rules)

- **CS-INV-1:** At most **one OPEN cash session per branch** at any time.
- **CS-INV-2:** CashMovements are immutable and append-only.
- **CS-INV-3:** A CLOSED or FORCE_CLOSED session cannot accept new movements.
- **CS-INV-4:** Sale finalization requires an OPEN cash session (Capstone I product rule).
- **CS-INV-5:** Cash movement creation must be idempotent.
- **CS-INV-6:** Reconciliation is a result, not a mutation of history.
- **CS-INV-7:** A cash refund movement (e.g., `REFUND_CASH`) must reference an approved void/refund and must be idempotent per the sale anchor (e.g., `(branch_id, sale_id)`).
- **CS-INV-8:** A cash refund movement must be recorded only to an OPEN cash session. It must never be appended to a CLOSED/FORCE_CLOSED session.

---

## 8. Domain Events

- CASH_SESSION_OPENED
- CASH_MOVEMENT_RECORDED
- CASH_SESSION_CLOSED
- CASH_SESSION_FORCE_CLOSED
- CASH_RECONCILIATION_COMPUTED

Events are used for:
- audit logging
- reporting projections
- downstream notifications

---

## 9. Read Model vs Write Model

### Write Model
- Enforces invariants and state transitions.
- Accepts commands and emits domain events.
- Guarantees atomicity for movements related to orders and voids.

### Read Model
- Current open session per branch.
- Movement list for reconciliation.
- X-report (session-level).
- Z-summary (day-level, derived).

---

## 10. Offline, Concurrency & Idempotency

- Cash session writes may be **queued offline** but:
  - must replay in order,
  - must fail deterministically if session is no longer OPEN.
- Every replayable command requires an idempotency key.
- Server is authoritative for session state and acceptance.

Offline sync **must not bypass invariants**.

---

## 11. Failure Modes & Recovery

| Scenario | Handling |
|----|----|
| Duplicate movement replay | Idempotency returns existing result |
| Sale finalize without open session | Rejected |
| Refund/void execution after session closed | Rejected |
| Network drop during movement | Safe retry |
| Forced close | Session ends, history preserved |

---

## 12. Out of Scope

- Multiple concurrent sessions per branch
- Register-level drawers
- Denomination-level counting
- Bank settlement automation

---

## 13. Notes & Future Evolution

- Multi-register support via Register domain
- Denomination tracking and analytics
- Session handover workflows
- Automated variance alerts

---

## 14. Derived Artifacts (Next Step)

- Patch CashSession ModSpec (single source of truth)
- Simplify backend implementation
- Align Reporting domain with this lifecycle
  
