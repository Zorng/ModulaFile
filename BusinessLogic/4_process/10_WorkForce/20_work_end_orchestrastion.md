# 20 — Work End Orchestration

## Purpose

This process describes how a staff member **finishes their work session** and how the system ensures the workplace is left in a clean, safe, and accountable state.

> Note: filename contains a typo (`orchestrastion`). Kept stable to avoid breaking existing links.

This is the counterpart of:
- `10_work_start_end_orchestration.md`

Together they form the full **Work Session Lifecycle**.

This orchestration connects:

- Attendance (actual work session)
- Access Control (who may perform closing actions)
- POS Operations (cash session + device responsibility)
- Staff Capacity (active staff slots)
- Work Review (future interpretation)

---

## Why this process exists

Ending work is not simply “clocking out”.

In a real store, when someone finishes work:

- Responsibility must be released
- Shared equipment must not be left in unsafe states
- Cash handling must be properly closed
- The system must stop counting the staff as “working”

Without a structured work-end process:

- Cash drawers may remain open overnight
- POS sessions may remain orphaned
- Staff capacity may be consumed incorrectly
- Attendance data becomes unreliable

This orchestration ensures the system transitions safely from:

ACTIVE WORK → NO ACTIVE WORK

---

## High-level outcome

When work ends successfully:

1. Responsibilities are resolved or transferred  
2. Attendance is closed with optional location verification  
3. Staff capacity is released  
4. Work duration becomes available for evaluation  

---

## Trigger

Primary trigger:
- Staff taps **End Work / Check Out**

Secondary triggers:
- Manager forced checkout
- Automatic checkout (future)

---

## Preconditions

The staff member must:

- Be authenticated
- Have selected tenant and branch
- Have an **ACTIVE attendance session**

If no active attendance exists → process stops.

Domain involved:
- Attendance

---

## Orchestration Overview

Work cannot end until **responsibilities are cleanly released**.

The system protects the business before closing attendance.

---

## Step 0 — Idempotency Gate (Process Layer)

Apply the platform idempotency gate:
- `action_key = attendance.endWork`
- `idempotency_key = client_op_id` (request-level key)

If the attendance session is already closed for this key:
- return success (no duplicate checkout)

Reference:
- `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`

---

## Step 1 — Validate active attendance

System verifies:

- Active attendance exists
- Attendance belongs to current tenant + branch

If validation fails → stop process.

Domain:
- Attendance

---

## Step 2 — Check unresolved responsibilities

Before checkout is allowed, the system verifies the staff has no **active responsibilities**.

Typical responsibilities:

- Open cash session
- Active POS device session
- Ownership of ongoing operations

If responsibilities exist:

Checkout is **blocked**.

The staff must:
- Close responsibilities, or
- Transfer them to another staff member

Why this matters:

Work must end **cleanly**.  
A workplace cannot be left in an unsafe or ambiguous state.

Domains involved:
- POS Operations
- Access Control

---

## Step 3 — Capture checkout evidence

The Attendance domain now captures **end-of-work evidence**.

The system records:

- checkout timestamp
- optional location verification (capability gated)

Location verification behaviour depends on:

`attendance.location_verification_mode`

Possible modes:

| Mode                 | Checkout behaviour            |
| -------------------- | ----------------------------- |
| disabled             | Record time only              |
| checkin_only         | Record time only              |
| checkin_and_checkout | Capture and evaluate location |

### Location verification result

Checkout may produce:

- MATCH — near workplace
- MISMATCH — far from workplace
- UNKNOWN — permission denied / weak signal / branch location not configured / disabled

The system records evidence only.  
It does not block checkout due to GPS issues.

Domain:
- Attendance

---

## Step 4 — Close attendance session

Attendance transitions:

ACTIVE → COMPLETED

This finalizes:

- work end time
- attendance duration
- optional checkout verification evidence

Attendance becomes immutable historical data.

Domain:
- Attendance

---

## Step 5 — Release staff work capacity

Once attendance is closed:

- Staff is no longer considered **actively working**
- Operator seat capacity is freed
- Real-time presence updates immediately

This supports the **Concurrent Operator Seat model** (Subscription & Entitlements).

Domains:
- Staff Capacity / Licensing
- Access Control

---

## Step 6 — Emit StaffWorkEnded event

System emits:

`StaffWorkEnded`

This event feeds:

- Shift vs Attendance Evaluation
- Work Review
- Reporting
- Notifications (future)

This event marks the official end of work.

---

## Failure scenarios

### Open cash session

If staff attempts checkout while a cash session is open:

System response:
- Checkout blocked
- Staff guided to close or transfer session

Reason:
Cash responsibility cannot disappear.

---

### Manager forced checkout

Managers/Admins may force checkout when:

- Staff forgot to check out
- Staff left workplace
- Device lost connection

System records:

- Who forced the checkout
- When correction occurred

This preserves accountability and auditability.

---

## Resulting state

After successful orchestration:

- Attendance session is completed
- Staff is no longer actively working
- Workplace responsibility is cleared
- Work data becomes ready for evaluation

---

## Relationship to other processes

Paired with:
- `10_work_start_end_orchestration.md`

Feeds into:
- 30_shift_vs_attendance_evaluation.md

This completes the **Work Session Lifecycle**.
