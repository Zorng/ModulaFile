# Identity & HR Edge Case Sweep

## Purpose

This document captures **explicit edge cases** discovered across the Identity & HR domains and processes.
Its goal is to:
- prevent silent assumptions,
- align team expectations,
- distinguish **March delivery guarantees** vs **future enhancements**.

This document is a **design safety net**, not a requirements explosion.

---

## Scope Covered

Domains and processes included:
- Tenant Membership
- Staff Profile & Assignment
- Shift (Planned Work)
- Attendance (Actual Work)
- Work Review
- Work Start / End Orchestration

Out of scope:
- Payroll
- Labor law enforcement
- Subscription billing implementation

---

## Legend

- **Owner**: where the decision belongs
- **March**: must be handled or clearly defined for March delivery
- **Later**: intentionally deferred

---

## 1. Tenant Membership Edge Cases

### 1.1 Removing the Last Owner
- **Scenario**: Admin attempts to remove or disable the last OWNER.
- **Expected Behavior**: Operation is rejected.
- **Owner**: Tenant Membership domain + Access Control
- **March**: Yes

### 1.2 Member Has Multiple Responsibilities
- **Scenario**: Same person is both ADMIN and operational STAFF.
- **Expected Behavior**:
  - Membership type remains singular.
  - Operational ability comes from Staff Profile existence.
- **Owner**: Tenant Membership + Staff Profile
- **March**: Yes

### 1.3 Removing a Member with History
- **Scenario**: Staff leaves after months of work.
- **Expected Behavior**:
  - Membership marked REMOVED.
  - History preserved.
- **Owner**: Tenant Membership
- **March**: Yes

---

## 2. Staff Profile & Assignment Edge Cases

### 2.1 Staff Assigned to Zero Branches
- **Expected Behavior**: START_WORK blocked until assignment exists.
- **Owner**: Staff Profile + Orchestration
- **March**: Yes

### 2.2 Staff Assigned to Multiple Branches
- **Expected Behavior**: Allowed if capability permits; branch must be selected.
- **Owner**: Staff Profile + Access Control
- **March**: Yes

### 2.3 Suspended / Archived Staff
- **Expected Behavior**: START_WORK denied.
- **Owner**: Staff Profile + Orchestration
- **March**: Yes

---

## 3. Shift (Planned Work) Edge Cases

### 3.1 Overlapping Planned Shifts
- **Expected Behavior**: Warn or block (policy decision).
- **Owner**: Shift
- **March**: Warn acceptable

### 3.2 Editing Recurring Shifts Mid-Week
- **Expected Behavior**: Apply prospectively.
- **Owner**: Shift
- **March**: Yes

---

## 4. Attendance Edge Cases

### 4.1 Duplicate Check-In
- **Expected Behavior**: Idempotent; return ACTIVE record.
- **Owner**: Attendance + Orchestration
- **March**: Yes

### 4.2 Forgotten Check-Out
- **Expected Behavior**: Remains ACTIVE; manual fix.
- **Owner**: Attendance
- **March**: Partial

### 4.3 Location Permission Denied
- **Expected Behavior**: Attendance created; attestation UNKNOWN.
- **Owner**: Attendance
- **March**: Yes

### 4.4 Branch Location Not Configured
- **Expected Behavior**: Attendance created; attestation UNKNOWN.
- **Owner**: Attendance + Branch
- **March**: Yes

---

## 5. Work Start / End Orchestration Edge Cases

### 5.1 Capacity Race Condition
- **Expected Behavior**: Best-effort atomic check.
- **Owner**: Orchestration
- **March**: Partial

### 5.2 Ending Work with Open Responsibilities
- **Expected Behavior**: End-work blocked or requires resolution.
- **Owner**: Orchestration + POS
- **March**: Must define

### 5.3 Duplicate End-Work
- **Expected Behavior**: Idempotent success.
- **Owner**: Orchestration
- **March**: Yes

---

## 6. Work Review Edge Cases

### 6.1 Multiple Attendance Segments
- **Expected Behavior**: Merge or collective compare.
- **Owner**: Work Review
- **March**: Partial

### 6.2 Missing Checkout
- **Expected Behavior**: INCOMPLETE_RECORD.
- **Owner**: Work Review
- **March**: Yes

---

## Summary

- Domains are structurally sound.
- Most risk lies in orchestration and policy clarity.
- March delivery is feasible with explicit degradation rules.

_End of document_
