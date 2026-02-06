# Shift Domain — Planned Work

## Domain Name
Shift (Planned Work)

## Domain Type
HR / Planning Domain

## Status
Draft (Refined — supports recurring planning)

---

## Purpose

The Shift domain represents **planned work expectations**.

A shift describes **when** a staff member is expected to work and **where** that work is expected to take place. It captures intent and planning, not enforcement or reality.

This domain exists to support:
- predictable operations,
- clear expectations for staff and managers,
- later evaluation of attendance and time-respecting behavior.

---

## Why This Exists (Story Traceability)

This domain is derived directly from:

- **Anchor A2 — Planning Who Is Expected to Work and Where**  
  Planning exists before work begins and represents expectation, not guarantee.

- **Anchor A6 — Reviewing Attendance and Time Respecting**  
  Planned work must be comparable to actual work over time to enable fair review.

Without a Shift domain, the system cannot distinguish between:
- what was planned,
- and what actually happened.

---

## Core Concepts

### Shift Pattern (Recurring Planned Work)

A **Shift Pattern** represents a **recurring expectation** of work.

Examples:
- “Every Monday to Friday, 08:00–17:00 at Branch A”
- “Every Saturday, 18:00–23:00 at Branch B”

A shift pattern defines:
- which days of the week work is expected,
- the expected start and end times,
- the branch where work is expected,
- the staff member the pattern applies to.

Shift patterns express **predictability**, not obligation.

---

### Shift Instance (Specific Planned Work)

A **Shift Instance** represents a **specific planned occurrence** of work on a particular date.

Examples:
- “Wednesday, Sept 18, 08:00–17:00 at Branch A”
- “Sunday, Sept 22, 18:00–23:00 at Branch B”

Shift instances may be:
- derived from a recurring shift pattern, or
- created independently for one-off situations.

---

## Relationship Between Pattern and Instance

- A shift pattern may produce many shift instances over time.
- Shift instances may exist without a pattern (ad-hoc planning).
- Cancelling or updating a single instance does not necessarily change the underlying pattern.
- Updating a pattern affects future instances only.

This separation allows planning to be:
- stable for full-time staff,
- flexible for exceptions,
- explicit when changes occur.

---

## Key Attributes (Conceptual)

A shift pattern typically includes:
- `tenant_id`
- `staff_id`
- `branch_id`
- `days_of_week`
- `planned_start_time`
- `planned_end_time`
- `status` (ACTIVE / INACTIVE)

A shift instance typically includes:
- `shift_instance_id`
- `tenant_id`
- `staff_id`
- `branch_id`
- `date`
- `planned_start_time`
- `planned_end_time`
- `status` (PLANNED / UPDATED / CANCELLED)

---

## Invariants

- All shifts belong to exactly one tenant.
- All shifts are associated with exactly one staff member.
- All shifts are tied to exactly one branch.
- Planned start time must be earlier than planned end time.
- Shift patterns and instances describe **expectation**, not permission.
- Shifts do not grant access or authorize work.

---

## What Shift Is NOT

The Shift domain does **not**:
- decide whether work is allowed to start,
- block or permit attendance,
- enforce staff capacity limits,
- record actual check-in or check-out times,
- manage payroll, overtime, or labor rules.

Those concerns belong to other domains.

---

## Relationship to Other Domains

### Shift ↔ Staff Profile
- A shift references a staff member.
- Historical shifts remain valid even if staff status changes.

### Shift ↔ Branch
- A shift is planned at a specific branch.
- Branch identity is referenced but not owned here.

### Shift ↔ Attendance
- Attendance records actual work.
- Attendance may exist with or without a shift.
- Shifts may exist with no corresponding attendance.

### Shift ↔ Access Control
- Shifts do not grant permission.
- Authorization is evaluated independently.

### Shift ↔ Work Review
- Work Review compares shift expectations with attendance reality.
- Shift provides the baseline expectation.

---

## Reality Considerations

- Many staff follow stable, recurring schedules.
- Some staff work irregular or ad-hoc shifts.
- Plans may change close to execution time.
- The system records declared expectations, not informal agreements.

---

## Out of Scope

- Shift swapping or approvals
- Payroll calculations
- Labor law enforcement
- Scheduling optimization

---

## Summary

The Shift domain captures **what was expected to happen**, both in recurring patterns and specific dates.

By separating **shift patterns** from **shift instances**, the domain supports:
- predictable planning for full-time staff,
- flexibility for exceptions,
- accurate comparison with actual attendance later.

---

_End of Shift Domain_
