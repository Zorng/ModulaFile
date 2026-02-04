# Identity/HR ↔ POS Boundary Edge Cases

## Purpose

This document hardens the boundary between:
- **Identity & HR** (who is working, where, when, under what entitlement)
and
- **POS Operations** (cash session, sale finalization, receipts, device actions)

It captures edge cases that emerge only when **human work lifecycle** meets **cash + device + order workflows**.

This is Phase **2.5 completion** work.

---

## Modules / Domains in Scope

Identity & HR:
- Authentication
- Tenant Membership
- Staff Profile & Assignment
- Shift (planned)
- Attendance (actual)
- Work Start/End Orchestration
- Access Control (authorization gate)

POS Operations:
- Cash Session
- Sale
- Receipt / Printing
- Inventory deduction (indirect)
- Device Draft / Device operations (open drawer, printer)

Platform/Systems (relevant):
- Audit
- Offline sync (partial)
- Operational notifications (later)

---

## Boundary Principles (Hard Rules)

1. **Work is not the same as Cash Session**
   - Attendance = “person working”
   - Cash Session = “cash responsibility state”
   - They often align, but are not identical.

2. **You cannot erase responsibility by ending work**
   - If a staff has unresolved operational responsibilities, END_WORK must not silently “clean it up.”

3. **Branch context is mandatory for POS actions**
   - Most POS actions are branch-scoped.
   - Staff may have multi-branch membership, but each action must operate in one branch context.

4. **Authorization is checked at critical moments**
   - Access can change mid-shift; gate must re-evaluate on sensitive actions.

---

## Edge Case Catalog

### B1 — Ending Work While Cash Session Is Still Open (Most Important)
- **Scenario**: Staff tries END_WORK but they have an ACTIVE cash session (drawer responsibility).
- **Risk**: cash drawer left open; no accountability.
- **Expected Behavior (March)**:
  - END_WORK is **blocked** with clear message:
    > “You must close your cash session before ending work.”
  - Only manager/owner may override (optional later).
- **Owner**: Work End Orchestration + Cash Session process
- **March**: Must define & enforce

---

### B2 — Cash Session Open Without Attendance
- **Scenario**: Cash session exists but attendance is missing (e.g., system bug, offline failure).
- **Expected Behavior (March)**:
  - Cash session remains valid (do not auto-close).
  - System flags as **INCONSISTENT_STATE** for audit.
  - Manager can manually correct attendance later.
- **Owner**: Audit + Cash Session + Attendance correction process
- **March**: Must degrade safely

---

### B3 — Attendance Active Without Cash Session
- **Scenario**: Staff is checked-in but has no cash session.
- **Expected Behavior (March)**:
  - Allowed. Many roles work without cash.
  - POS may block actions that require drawer access until cash session is opened.
- **Owner**: POS UI/Access Control + Cash Session module
- **March**: Must support

---

### B4 — Switching Branch Mid-Shift
- **Scenario**: Staff checks-in at Branch A, then attempts to operate Branch B.
- **Expected Behavior (March)**:
  - Staff must END_WORK at Branch A before START_WORK at Branch B.
  - If you allow same-day split shifts, this becomes two attendance records.
- **Owner**: Work Start Orchestration + Access Control
- **March**: Prefer strict rule (simpler)

---

### B5 — Logged In on Two Devices at Once
- **Scenario**: Same staff identity uses two tablets simultaneously.
- **Risk**: duplicate sales, conflicting cash responsibility.
- **Expected Behavior (March)**:
  - Allowed for viewing/low-risk actions, but:
    - START_WORK should be idempotent (one ACTIVE attendance).
    - Cash session should be single-active per staff+branch.
  - Optionally show “active on another device” warning.
- **Owner**: Authentication session mgmt + Attendance + Cash Session
- **March**: Must not corrupt state

---

### B6 — Permission/Role Changes Mid-Shift
- **Scenario**: Manager revokes cashier permission while cashier is still working.
- **Expected Behavior (March)**:
  - Access Control re-check on each sensitive action.
  - Already-open screens may fail on next action with:
    > “Your access has changed. Please contact manager.”
- **Owner**: Access Control middleware + UX
- **March**: Must define

---

### B7 — Staff Archived While Still Working
- **Scenario**: Owner archives staff while they are checked-in and/or have open cash session.
- **Expected Behavior (March)**:
  - Archiving should be blocked if:
    - ACTIVE attendance exists OR
    - ACTIVE cash session exists
  - Or archiving triggers a “must resolve first” workflow.
- **Owner**: Staff Management + Orchestration
- **March**: Strongly recommended

---

### B8 — Sale Finalization Requires a Valid Work Context?
- **Scenario**: Device has an active cart; staff tries finalize sale but is not checked-in.
- **Expected Behavior (March)**:
  - **Finalize sale requires:**
    - authenticated identity
    - ACTIVE tenant membership
    - branch context
  - Attendance requirement is a **policy choice**:
    - For March: allow finalize sale if staff is authenticated and authorized, even if attendance is missing.
    - But log audit signal: “sale processed without attendance.”
- **Owner**: Sale orchestration + Audit + policy decision
- **March**: Must define clearly

---

### B9 — Cash Movement Recording vs Attendance
- **Scenario**: Sale finalization records cash movement but attendance ends right after.
- **Expected Behavior**:
  - Cash movement belongs to cash session / sale record, not attendance.
  - Attendance is context only; it must not be required for historical correctness.
- **Owner**: Cash Session + Sale
- **March**: Must preserve financial correctness

---

### B10 — Offline Mode and Reconciliation
- **Scenario**: Device works offline; attendance/cash session events sync later.
- **Expected Behavior (March)**:
  - Events must be idempotent and order-tolerant.
  - Conflicts must be auditable.
  - If too complex for March, scope offline for HR features can be “best effort.”
- **Owner**: Offline Sync + Audit
- **March**: Partial (define constraints)

---

## Policies to Decide Explicitly (March Decisions)

These are “must choose one” to avoid ambiguity:

1) **END_WORK block if cash session open**: Block vs override vs auto-close  
→ Recommendation: **Block**.

2) **Finalize sale requires attendance?**  
→ Recommendation: **No hard block** (for resilience), but **audit flag**.

3) **Archiving staff with active responsibilities**  
→ Recommendation: **Block archive** until resolved.

4) **Branch switching mid-shift**  
→ Recommendation: **Require end/start** (two attendance records).

---

## Outputs / Next Patches

After locking this document, we should patch:
- Work Start/End Orchestration process to include cash-session dependency (B1, B4)
- Staff management modspec to handle archive guard (B7)
- Sale finalization orchestration to state attendance requirement (B8)
- Audit modspec to include inconsistency flags (B2, B8, B10)

---

_End of Identity/HR ↔ POS Boundary Edge Cases_
