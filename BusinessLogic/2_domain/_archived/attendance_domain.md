# Attendance Domain — Actual Work

## Domain Name
Attendance (Actual Work)

## Domain Type
HR / Operational Reality Domain

## Status
Draft (Derived from Anchor Stories A3, A4, and A6)

---

## Purpose

The Attendance domain represents **what actually happened during work**.

Attendance records the moments when a staff member **starts** and **ends** working at a branch. It captures reality as it occurred, regardless of what was planned.

This domain exists to:
- establish a reliable source of truth for actual work,
- support operational control during the day,
- enable later review of time respecting and behavior,
- free or consume operational capacity correctly.

---

## Why This Exists (Story Traceability)

This domain is derived directly from:

- **Anchor A3 — Starting Work at a Branch**  
  Work begins when a person actually arrives and starts working.

- **Anchor A4 — Ending Work and Leaving the System Clean**  
  Work ends explicitly and must leave the system in a consistent state.

- **Anchor A6 — Reviewing Attendance and Time Respecting**  
  Actual behavior must be reviewable and comparable to planned expectations.

Without an Attendance domain, the system cannot reliably know:
- who is working right now,
- when responsibility started or ended,
- or what actually happened over time.

---

## Core Concepts

### Attendance Record

An **Attendance Record** represents a continuous period of actual work by a staff member at a specific branch.

It answers the questions:
- who worked,
- where they worked,
- when they started,
- when they stopped.

An attendance record may or may not correspond to a planned shift.

---

## Key Attributes

An Attendance Record minimally contains:
- `attendance_id`
- `tenant_id`
- `staff_id`
- `branch_id`
- `actual_start_time`
- `actual_end_time` (nullable while working)
- `status` (ACTIVE, COMPLETED)
- `created_at`
- `updated_at`

---

## Invariants

- An attendance record always belongs to exactly one tenant.
- An attendance record is always associated with exactly one staff member.
- An attendance record is always tied to exactly one branch.
- A staff member can have **at most one ACTIVE attendance record at a time**.
- An attendance record always has a start time.
- An attendance record may end later than planned or earlier than planned.
- Attendance describes **reality**, not permission.

---

## Attendance Lifecycle

An attendance record typically moves through these states:

- **ACTIVE** — staff has started work and is currently working
- **COMPLETED** — staff has ended work

Attendance records are never deleted. They represent historical facts.

---

## What Attendance Is NOT

The Attendance domain does **not**:
- decide whether a staff member is allowed to start work,
- enforce branch access rules,
- enforce staff capacity limits,
- define what work is expected,
- manage payroll or compensation.

Those concerns belong to other domains.

---

## Relationship to Other Domains

### Attendance ↔ Shift (Planned Work)
- Shift defines what was expected.
- Attendance defines what actually happened.
- Attendance may exist without a corresponding shift.
- Shifts may exist with no corresponding attendance.

---

### Attendance ↔ Staff Profile
- Attendance references a staff member.
- Attendance remains valid even if staff status changes later.

---

### Attendance ↔ Branch
- Attendance always occurs at a specific branch.
- Branch identity is referenced, not owned.

---

### Attendance ↔ Access Control
- Access Control decides whether work may begin.
- Attendance records work **after** authorization has succeeded.

---

### Attendance ↔ Work Capacity / Licensing
- ACTIVE attendance contributes to concurrent staff capacity.
- COMPLETED attendance frees capacity.

---

### Attendance ↔ Work Review
- Attendance provides the raw data for review.
- Interpretation happens elsewhere.

---

## Reality Considerations

- Staff may start work without a planned shift.
- Staff may end work late, early, or unexpectedly.
- Mistakes happen (forgotten check-out).
- Manual correction may be required later (handled by separate processes).

The Attendance domain records facts, even when those facts are imperfect.

---

## Out of Scope

- Shift planning
- Authorization logic
- Capacity policy definition
- Payroll and labor law enforcement
- Automated correction or judgment of behavior

---

## Summary

The Attendance domain captures **what actually happened** during work.

It provides:
- a clear boundary for responsibility,
- reliable signals for operational control,
- and factual input for later review.

By separating Attendance from planning and permission, the system remains fair, auditable, and adaptable.

---

_End of Attendance Domain_
