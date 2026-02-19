# Edge Case Contract — Identity & HR

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: Identity & HR context (internal to workforce + identity membership rules)
- **Primary Audience**: Backend, Frontend, QA
- **Owner(s)**: Tenant Membership, Staff Profile & Assignment, Shift, Attendance, Work Review, Work Start/End Orchestration
- **Last Updated**: 2026-02-08
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred

---

## Purpose
This document captures **explicit edge cases** discovered across the Identity & HR domains and processes.
It exists to prevent silent assumptions and to keep the team aligned on what must work by **March** versus what is intentionally deferred.

---

## Scope
### In-scope
- Tenant Membership lifecycle edge cases
- Staff Profile & branch assignment edge cases
- Shift planning (including recurring changes)
- Attendance capture (including location attestation behaviors)
- Work Review interpretation of records
- Work Start/End orchestration idempotency and capacity checks

### Out-of-scope
- Payroll
- Labor law enforcement
- Subscription billing implementation

---

## Definitions / Legend
- **Owner**: where the decision belongs (domain/process)
- **March**: must be handled or clearly defined for March delivery
- **Later**: intentionally deferred

---

## Edge Case Catalog

### EC-IDH-01 — Removing the Last Owner
- **Scenario**: Admin attempts to remove or disable the last OWNER.
- **Trigger**: Membership update request would result in zero owners.
- **Expected Behavior**: Operation is rejected.
- **Owner**: Tenant Membership + Access Control
- **March**: Yes

### EC-IDH-02 — Member Has Multiple Responsibilities
- **Scenario**: Same person is an owner/admin and also works shifts as staff.
- **Trigger**: Membership has `membership_kind = OWNER` and/or `role_key = ADMIN`, and Staff Profile exists.
- **Expected Behavior**:
  - Membership remains a single record per tenant with `membership_kind` + `role_key`.
  - Staff Profile exists to support operational work (branch assignment + display context like job title).
  - Authorization is evaluated from `role_key` + branch assignment; job title is not an authorization input.
- **Owner**: Tenant Membership + Staff Profile & Assignment + Access Control
- **March**: Yes

### EC-IDH-03 — Removing a Member with History
- **Scenario**: Staff leaves after months of work.
- **Trigger**: Owner/admin removes membership.
- **Expected Behavior**:
  - Membership marked REVOKED.
  - History preserved.
- **Owner**: Tenant Membership
- **March**: Yes

### EC-IDH-04 — Staff Assigned to Zero Branches
- **Scenario**: Staff is created but assigned to no branches.
- **Trigger**: START_WORK attempted.
- **Expected Behavior**: START_WORK blocked until assignment exists.
- **Owner**: Staff Profile + Work Start orchestration
- **March**: Yes

### EC-IDH-05 — Staff Assigned to Multiple Branches
- **Scenario**: Staff is assigned to multiple branches.
- **Trigger**: Staff starts work or performs branch-scoped actions.
- **Expected Behavior**: Allowed; branch must be selected.
- **Owner**: Staff Profile + Access Control
- **March**: Yes

### EC-IDH-06 — Revoked Staff Tries to Work
- **Scenario**: Membership and/or staff profile is REVOKED but the person tries to work.
- **Trigger**: START_WORK attempted.
- **Expected Behavior**: START_WORK denied.
- **Owner**: Staff Profile + Work Start orchestration
- **March**: Yes

### EC-IDH-07 — Overlapping Planned Shifts
- **Scenario**: Shift planner creates overlapping shifts.
- **Trigger**: Shift create/update.
- **Expected Behavior**: Warn or block (policy decision).
- **Owner**: Shift
- **March**: Warn acceptable

### EC-IDH-08 — Editing Recurring Shifts Mid-Week
- **Scenario**: Planner edits a recurring shift mid-week.
- **Trigger**: Recurrence edit request.
- **Expected Behavior**: Apply prospectively.
- **Owner**: Shift
- **March**: Yes

### EC-IDH-09 — Duplicate Check-In
- **Scenario**: Staff taps check-in twice or retries due to network.
- **Trigger**: Duplicate START_WORK/Check-in request.
- **Expected Behavior**: Idempotent; return ACTIVE record.
- **Owner**: Attendance + Work Start orchestration
- **March**: Yes

### EC-IDH-10 — Forgotten Check-Out
- **Scenario**: Staff forgets to end work.
- **Trigger**: Attendance remains ACTIVE beyond expected.
- **Expected Behavior**: Record remains ACTIVE; manual fix later.
- **Owner**: Attendance
- **March**: Partial

### EC-IDH-11 — Location Permission Denied
- **Scenario**: Device denies GPS permission.
- **Trigger**: Check-in with location feature enabled.
- **Expected Behavior**: Attendance created; attestation UNKNOWN.
- **Owner**: Attendance
- **March**: Yes

### EC-IDH-12 — Branch Location Not Configured
- **Scenario**: Branch has no configured location.
- **Trigger**: Check-in with location feature enabled.
- **Expected Behavior**: Attendance created; attestation UNKNOWN.
- **Owner**: Attendance + Branch
- **March**: Yes

### EC-IDH-13 — Capacity Race Condition
- **Scenario**: Two staff start work at the same time and cross capacity limit.
- **Trigger**: Concurrent START_WORK.
- **Expected Behavior**: Best-effort atomic check.
- **Owner**: Work Start orchestration
- **March**: Partial

### EC-IDH-14 — Ending Work with Open Responsibilities
- **Scenario**: Staff tries to end work while some responsibilities remain open.
- **Trigger**: END_WORK request.
- **Expected Behavior**: End-work blocked or requires resolution.
- **Owner**: Work End orchestration + POS boundary rules
- **March**: Must define (see boundary contract)

### EC-IDH-15 — Duplicate End-Work
- **Scenario**: Staff taps end-work twice or retries due to network.
- **Trigger**: Duplicate END_WORK.
- **Expected Behavior**: Idempotent success.
- **Owner**: Work End orchestration
- **March**: Yes

### EC-IDH-16 — Multiple Attendance Segments
- **Scenario**: Staff works in multiple segments in one day (breaks / split shifts).
- **Trigger**: Multiple attendance records.
- **Expected Behavior**: Merge or collectively compare (policy).
- **Owner**: Work Review
- **March**: Partial

### EC-IDH-17 — Missing Checkout Interpreted in Review
- **Scenario**: Work review runs on a record that never ended.
- **Trigger**: Review job evaluates ACTIVE/unfinished attendance.
- **Expected Behavior**: Mark INCOMPLETE_RECORD.
- **Owner**: Work Review
- **March**: Yes

### EC-IDH-18 — Location Mismatch (Outside Allowed Radius)
- **Scenario**: Device location is available but clearly outside the branch allowed radius.
- **Trigger**: Check-in (or check-out) with location feature enabled and workplace location configured.
- **Expected Behavior**: Attendance is created/closed; attestation MISMATCH is recorded; work is not blocked by GPS.
- **Owner**: Attendance + Work Review
- **March**: Yes

---

## Summary
- Most risk lies in orchestration, idempotency, and cross-boundary responsibilities.
- March delivery is feasible with explicit degradation rules (UNKNOWN/MISMATCH attestation, partial atomicity).
- Cross-boundary rules are centralized in `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md`.

_End of Identity & HR edge case contract_
