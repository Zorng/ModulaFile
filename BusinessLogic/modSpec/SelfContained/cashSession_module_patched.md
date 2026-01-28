# Cash Session & Reconciliation Module (Feature Module)

**Version:** 1.7  
**Status:** Patched (Cash sessions required for sales; branch system-provisioning alignment)  
**Module Type:** Feature Module  
**Depends on:** Auth & Authorization (Core), Tenant & Branch Context (Core), Policy & Configuration (Core), Audit Logging (Core), Sync & Offline Support (Core)  
**Related Modules:** Sale (cash tender), Reporting (X/Z), Staff Attendance (independent), Branch Management (Core concept; system-provisioned branches)

---

## Purpose

The Cash Session module ensures **cash integrity per shift** by tracking:
- opening float → cash movements (cash sales, paid out, refunds, adjustments) → closing count → variance

It provides operational control via:
- **X Report** (in-session snapshot)
- **Z Report** (final session close report)

Cash Session is **not an attendance mechanism** and is **not shift-gated**. Shift rules belong to the Staff Attendance module.

---

## Scope (Capstone I)

Included:
- Start/Open cash session (opening float)
- End/Close cash session (counted cash, variance)
- Policy-controlled cash movements:
  - Cash Sale tender attachment (system)
  - Paid Out (on/off)
  - Cash Refund approval (on/off)
  - Manual Cash Adjustment (on/off)
- X / Z reporting
- Offline-first staging + sync
- Audit logging for all actions
- **Force-close** by Manager/Admin (no takeover)

Excluded (Future Work):
- Register/device-owned sessions
- Cash drawer hardware control
- Multiple terminals linked to one drawer/hardware hub
- Safe drop / cash lift workflows
- Automated anomaly detection

---

## Key Concepts

- **Branch (System-Provisioned):** Branches are created by the system when a tenant is provisioned (e.g., initial subscription creates tenant + first branch; adding branches via subscription creates additional branches).  
  - Users **cannot** create branches manually in Capstone I.  
  - Cash Sessions always operate within an existing **branch context**.
- **Cash Session:** A time-bounded record representing cash handling for a branch shift by a user (Capstone I).
- **Cash Movements:** Append-only entries affecting expected cash.
- **Force Close:** Manager/Admin closes an active session when the opener cannot (e.g., left, forgot, device lost), without editing history.

---


## Aggregate & Data Rules (Write Model)

### Aggregate Root: CashSession
- Identifiers: `cash_session_id`, `branch_id`
- State: `OPEN | CLOSED | FORCE_CLOSED`
- Timestamps: `opened_at`, `closed_at`
- Metadata: `opened_by_user_id`, `closed_by_user_id`, `force_close_reason` (required if FORCE_CLOSED)

**Allowed transitions**
- `OPEN → CLOSED`
- `OPEN → FORCE_CLOSED`

**Forbidden transitions**
- `CLOSED → OPEN`
- `FORCE_CLOSED → CLOSED` (or any further changes)

### Child Entity: CashMovement (append-only ledger)
- Required fields:
  - `cash_movement_id`
  - `cash_session_id`
  - `branch_id`
  - `movement_type` (e.g., `CASH_SALE`, `PAID_IN`, `PAID_OUT`, `REFUND`, `ADJUSTMENT`)
  - `amount` (Money: currency + value; USD/KHR supported)
  - `created_at`
  - `source_reference` (e.g., `order_id`, `void_id`, or `manual_ref`)
  - **`idempotency_key`** (required for offline replay / safe retries)
- Rules:
  - Movements are **immutable** and **never edited/deleted**
  - Movements can be recorded **only** while the parent session is `OPEN`
  - Reconciliation is computed from movements + counted snapshot; it **must not** rewrite movement history

## Session Ownership Model

### Capstone I (Current): **Branch Accountability**
- A cash session represents the **cash accountability window for a branch**.
- **Invariant:** at most **one OPEN cash session per branch** at any time.
- `opened_by_user_id` records who opened the session (for audit), but **does not** determine uniqueness.
- Closing actor may be the opener or an authorized Manager/Admin (per auth policy).

### Future Work (Hardware / Registers)
- Introduce **register/device/drawer** concepts.
- Sessions may become **register-owned**, while still remaining branch-scoped for reporting.

> This spec optimizes for offline-first mobile/web usage now, while keeping a path to register-based operations later.

## Policy Scope Clarification

All cash-session-related policies are **branch-scoped**.

That means:
- Policy values are resolved using **(tenant_id, branch_id)**.
- Two branches under the same tenant may legally have different cash-session rules (e.g., allow paid-out, refund approval, manual adjustments).

**Product rule (Capstone 1):** cash sessions are required for sales/cart creation, so “require session for sales” is not configurable.

The policy keys remain the same (see “Policies Used”), but **their values must be fetched for the active branch**.

---

## Use Cases

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

**Actors:** Cashier (if allowed by role rules), Manager, Admin

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

> Note: There is **no** backend policy key for “allow paid in” in the current policy schema. If you want it policy-controlled later, it must be added explicitly (and would be branch-scoped).

---

### UC-5: Paid Out

**Actors:** Cashier (if allowed), Manager, Admin

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE
- Session OPEN
- **Branch-scoped policy** `cashAllowPaidOut = true`

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
- **Branch-scoped policy** `cashRequireRefundApproval = true`
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
- **Branch-scoped policy** `cashAllowManualAdjustment = true`

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

**Main Flow:**
1. Actor taps “Close Session”.
2. Inputs counted cash (USD/KHR).
3. System computes expected cash and variance.
4. Session marked CLOSED.
5. Z report generated.

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

**Actors:** Manager, Admin

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE
- A session exists in OPEN state (opened by another user or abandoned)
- Manager/Admin is authorized in branch scope

**Main Flow:**
1. Manager/Admin selects the OPEN session.
2. System requires a force-close reason (required) and optional note.
3. Manager/Admin inputs counted cash (or marks “not counted” if supported).
4. System closes the session as FORCE_CLOSED (or CLOSED with `closure_type`).
5. System computes variance.
6. Audit log records force-close action.

**Postconditions:**
- Session is no longer OPEN.
- Closure is traceable (who, when, why).
- Z report available (with force-close flag).

**Key Rule:**
- Force-close does **not** edit any prior movements or timestamps. It is an operational override only.

---

### UC-10: View X Report

**Actors:** Cashier, Manager, Admin

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE
- Session OPEN

**Main Flow:**
1. Actor opens X report.
2. System shows opening float, movement totals, expected cash.

**Postconditions:**
- Read-only.

---

### UC-11: View Z Report

**Actors:** Cashier, Manager, Admin

**Preconditions:**
- Branch context is resolved
- Branch is ACTIVE (or recently frozen; read-only access still allowed if tenant rules permit)
- Session CLOSED / FORCE_CLOSED

**Main Flow:**
1. Actor opens Z report.
2. System shows final totals, counted cash, variance, closure metadata.

**Postconditions:**
- Read-only.

---

### UC-12: Offline Operation & Sync

**Actors:** System

**Preconditions:**
- Network unavailable or unstable
- Branch context is known locally (cached)

**Main Flow:**
1. Open/close and movements can be staged locally as queued commands with idempotency keys.
2. On reconnect, the client replays queued commands **in order**; server enforces invariants and returns deterministic outcomes.
3. If the session is no longer OPEN (closed/force-closed), movement commands are rejected; UI must surface the reason and next action.

**Postconditions:**
- No duplicate sessions/movements (idempotency).
- Totals remain consistent with server truth.
- Rejections do not corrupt local ledgers; they produce actionable UI feedback.

---

## Policies Used (Read-only from Policy Module)

All policy keys below must match the backend schema exactly.  
**Policy values are resolved per (tenant_id, branch_id).**

- *(product-enforced)* require session for sales/cart creation (always true in Capstone 1)
- `cashAllowPaidOut` (boolean)
- `cashRequireRefundApproval` (boolean)
- `cashAllowManualAdjustment` (boolean)

> No shift policy is enforced here. Shift enforcement belongs to Staff Attendance.

---

## Requirements

- R1: Sessions operate only within an existing, system-provisioned branch (no user-created branches in Capstone I)
- R2: At most **one OPEN cash session per branch** at any time
- R3: Cash-based order finalization requires an **OPEN cash session for the branch**
- R4: Movements are append-only; no edits or deletes
- R5: Movement creation must be **idempotent** (movement-level idempotency key required)
- R6: Managers/Admins can force-close sessions with required reason
- R7: X/Z reports must reflect movements and reconciliation variance (expected vs counted)
- R8: Offline-first with idempotent sync; server remains authoritative
- R9: All actions logged via Audit Logging
- R10: Branch frozen/suspended blocks creating/updating sessions and movements (read-only access may remain)

---

## Acceptance Criteria

- AC1: User can open a session with opening float in an ACTIVE branch.
- AC2: Cash sale increases expected cash via recorded movement.
- AC3: User can close session and system computes variance.
- AC4: Manager/Admin can force-close an abandoned session; reason is required.
- AC5: No role can edit historical movement amounts or timestamps.
- AC6: X report works during open session; Z report works after close.
- AC7: Offline-staged actions sync without duplication.
- AC8: If branch is frozen, session creation/movements are blocked.

---

## Audit Events Emitted

These are emitted to the Audit Logging module. Some correspond 1:1 with domain events.

- CASH_SESSION_OPENED
- CASH_MOVEMENT_RECORDED
- CASH_SESSION_CLOSED
- CASH_SESSION_FORCE_CLOSED
- CASH_RECONCILIATION_COMPUTED
- CASH_TENDER_ATTACHED_TO_SALE
