# Cash Session Module — Feature Module Spec (Capstone 1)

**Version:** 1.3 (Patched)  
**Status:** Clarified to distinguish operational reconciliation (X/Z) from analytics reporting.  
**Module Type:** Feature Module  
**Primary Domain:** POS Operations → Cash Session  
**Depends on:** Authentication, Access Control, Tenant & Branch Context, Sale/Order, Audit Logging  
**Collaborates with (Cross-Module):** Sale/Order (Finalize Sale, Void Sale), Reporting (read-only consumption)

---

## 1. Purpose

The Cash Session module manages **cash drawer state and reconciliation** for a branch.

It exists to ensure:
- cash movements are **auditable**
- drawer state is **explicit and bounded in time**
- staff can reconcile expected vs actual cash confidently

This module models **operational cash handling**, not long-term financial analytics.

---

## 2. Scope & Boundaries

### Includes (Operational)
- Opening and closing cash sessions
- Recording cash movements tied to finalized sales
- Manual cash adjustments (cash-in / cash-out with reasons)
- Operational reconciliation views (X and Z)
- Enforcing one OPEN cash session per branch

### Excludes
- Payment gateway settlement
- Bank reconciliation
- Cross-session analytics and trends (Reporting module)
- Accounting exports

---

## 2.1 Subscription & Entitlements Integration (Billing Guard Rails)

Cash Session is part of the core POS bundle and is required for operational accountability (X/Z).

- If tenant subscription state is `PAST_DUE`:
  - cash session operations continue (authorization allows), but UX must warn.
- If tenant subscription state is `FROZEN`:
  - cash session write actions are blocked (cannot open/close or record manual movements),
  - historical reads (past sessions, X/Z artifacts) may remain available.

Enforcement is performed by Access Control using action metadata (scope + effect) and subscription state.

---

## 3. Core Concepts

### 3.1 Cash Session
A bounded period during which a physical cash drawer is active.

States:
- OPEN
- CLOSED

Each branch may have **at most one OPEN cash session** at a time.

### 3.2 Cash Movement (Ledger)
An append-only record representing a cash change:
- SALE_IN
- REFUND_CASH
- MANUAL_IN
- MANUAL_OUT

Cash movements are immutable and auditable.

---

## 4. Use Cases — Self-Contained Processes

### UC-1: Start Cash Session (Open)

**Actors:** Cashier, Manager, Admin

**Preconditions:**
- User is authenticated
- **Branch context is resolved** (branch exists and is accessible to the actor)
- Branch is **ACTIVE** (not frozen/suspended)
- No active cash session exists for this branch

**Main Flow:**
1. User navigates to Sale or taps “Start Cash Session”.
2. System prompts for opening float (USD/KHR).
3. User confirms.
4. System creates an OPEN cash session.
5. Audit log records session opened.

**Postconditions:**
- An OPEN cash session exists and is active for the branch.

---

### UC-2: Continue Active Cash Session

**Actors:** Cashier, Manager, Admin

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE
- An OPEN cash session exists for the branch

**Main Flow:**
1. User opens Sale.
2. System attaches eligible cash activity to the active session.

**Postconditions:**
- No state change.

---

### UC-3: Record Cash Tender From Sale (System)

**Actors:** System (triggered by Sale module)

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE
- Sale finalized with payment method = Cash
- An OPEN cash session must exist for that branch (Capstone I product rule)

**Main Flow:**
1. Sale finalizes with cash tender.
2. System derives an idempotency key from the sale (e.g., `sale_id` or `{sale_id}:{attempt}`) for safe retries.
3. System records a CASH_SALE movement linked to the sale using that idempotency key.
4. Reconciliation totals update via projections/read model (expected cash is derived).

**Postconditions:**
- Movement recorded exactly once (idempotent).
- Derived expected cash reflects the new movement.
- Movement visible in X/Z reports.
- Audit log records cash tender attachment.

**Error Flow:**
- If no session exists → sale/cart creation is blocked until a session is opened (Sale module behavior).

---

### UC-4: Cash Paid In (Optional Capability)

**Actors:** Cashier, Manager, Admin

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE
- Session is OPEN

**Main Flow:**
1. User selects “Add Cash / Paid In”.
2. Inputs amount (USD or KHR) and reason.
3. System records PAID_IN movement.

**Postconditions:**
- Reconciliation totals update via projections/read model (expected cash is derived).
- Action is audit logged.

> Note: Cash movements such as Paid In / Paid Out / Manual Adjustment are **not policy-controlled** in Capstone 1.
> They are governed by Access Control (RBAC) and must be fully auditable.
> If you want tenant/branch toggles later, add explicit policy keys (branch-scoped).

---

### UC-5: Paid Out

**Actors:** Cashier, Manager, Admin

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE
- Session OPEN

**Main Flow:**
1. User selects “Paid Out”.
2. Inputs amount and reason.
3. System records PAID_OUT movement.

**Postconditions:**
- Expected cash decreases.
- Action is audit logged.

---

### UC-6: Cash Refund (Approval-Controlled)

**Actors:** Manager, Admin

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE
- Session OPEN
- A refund/void approval exists (triggered by void/refund flow)

**Main Flow:**
1. Manager/Admin reviews refund/void approval request.
2. Approves or rejects.
3. If approved:
   - System records REFUND_CASH movement linked to sale.
   - Expected cash decreases.

**Postconditions:**
- Refund decision is recorded and auditable.
- No editing of past cash sale movements.

---

### UC-7: Manual Cash Adjustment

**Actors:** Manager, Admin

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE
- Session OPEN

**Main Flow:**
1. Actor inputs adjustment (+/-) and reason.
2. System records ADJUSTMENT movement.

**Postconditions:**
- Expected cash updated.
- Action is audit logged.

---

### UC-8: Close Cash Session (Normal Close)

**Actors:** Cashier, Manager, Admin

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE
- Session OPEN
- Actor has permission to close the session
- No unpaid tickets exist in the branch (pay-later orders must be settled or cancelled first)

**Main Flow:**
1. Actor taps “Close Session”.
2. Inputs counted cash (USD/KHR).
3. System computes expected cash and variance.
4. Session marked CLOSED.
5. Z report generated.
6. System emits OperationalNotifications (best-effort, idempotent):
   - ON-04 `CASH_SESSION_CLOSED:{branch_id}:{cash_session_id}` (awareness; optional March baseline)
   - include variance values in the notification body/payload so humans can evaluate

**Postconditions:**
- Session status = CLOSED
- Variance stored
- Z report available
- Action is audit logged

**Notes:**
- System does not auto-close sessions.
- Shift timing does not gate closing.

---

### UC-9: Force Close Cash Session (Manager/Admin Only)
**Actor:** Manager, Admin  
**Flow:** Enter amount + reason → append MANUAL_IN / MANUAL_OUT  
**Postconditions:** Ledger updated; visible in reconciliation

---

### UC-10: View X Report (Operational Snapshot)
**Actor:** Cashier (own session), Manager, Admin  
**When:** Cash session is OPEN  
**Purpose:**  
Provide a **real-time operational snapshot** of the current cash session.

Includes:
- opening float
- sum of cash movements so far
- expected cash-in-drawer at this moment

**Notes:**
- X Report is read-only
- X Report does not close the session
- X Report is part of **cash handling UX**, not analytics
- Access Control action key (branch-scoped): `cashSession.x.view`  
  - Cashier is limited to their own session(s) (Capstone 1 baseline).

---

### UC-11: View Z Report (Session Close Summary)
**Actor:** Cashier (own session), Manager, Admin  
**When:** Cash session is CLOSED  
**Purpose:**  
Provide a **final reconciliation summary** for the session.

Includes:
- opening float
- total cash movements
- counted cash
- variance
- closure metadata (who, when, reason)

**Notes:**
- Z Report represents the authoritative close of a cash session
- Z Report is immutable once created
- Z Report is an **operational artifact**, not an analytics report
- Access Control action key (branch-scoped): `cashSession.z.view`  
  - Cashier is limited to their own session(s) (Capstone 1 baseline).

---

## 5. Cross-Module Participation

### CM-1: Finalize Sale → Cash Movement
**Trigger:** Finalize Sale process  
**Behavior:**  
If payment includes cash:
- append SALE_IN cash movement
- idempotent per `(branch_id, sale_id)`

---

### CM-2: Reporting Read Contract
**Consumer:** Reporting module  
**Behavior:**  
Reporting may read:
- CLOSED cash sessions (Z summaries)
- cash movement ledgers

Reporting must not:
- mutate cash sessions
- reinterpret operational reconciliation logic

---

## 6. Idempotency & Invariants

- All critical cash session writes must pass the platform idempotency gate:
  - `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`
- Only one OPEN cash session per branch
- Cash movements are append-only
- Sale-based cash movements are idempotent per sale_id
- Closed sessions cannot be modified

---

## 7. Audit Events

- CASH_SESSION_OPENED
- CASH_SESSION_CLOSED
- CASH_MOVEMENT_RECORDED
- CASH_ADJUSTMENT_RECORDED
- CASH_X_REPORT_VIEWED (observational)
- CASH_Z_REPORT_VIEWED (observational)

Audit events for X/Z are **observational** and do not represent state changes.

---

## 8. Out of Scope (Capstone 1)

- Cash forecasting
- Multi-currency drawers
- Bank deposit workflows
- Advanced variance analytics
- Staff performance analytics

---

## 9. Design Notes

X and Z reports live in this module because they are:
- tied to a single cash session
- required for drawer operation
- expected by users as part of cash handling

Analytics across sessions belong to the Reporting module.

---

# End of Document
