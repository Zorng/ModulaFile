# Attendance Domain — Actual Work

## Domain Name
Attendance (Actual Work)

## Domain Type
HR / Operational Reality Domain

## Status
Draft (Refined — supports location confirmation and capability gating)

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

### Check-In Location Attestation (Optional Enrichment)

For March delivery, attendance check-in can optionally support **location confirmation** (GPS or equivalent signal).

A **Check-In Location Attestation** is the recorded evidence of where the device believed it was at the moment a staff member checked in, and how that compared to the expected workplace location (the branch).

**Important:** Location attestation is an *enrichment*, not the definition of attendance.  
Because Modula follows the philosophy “you only pay for what you use,” some tenants will not enable or purchase GPS-based attendance. Therefore:

- Attendance must work correctly **with or without** location attestation.
- If location attestation is not enabled, not available, or not permitted, attendance is still valid.

This concept exists because real-world location signals are imperfect. The system must be able to record:
- what evidence was available,
- what comparison was performed,
- and what the outcome was,
without pretending it is always perfectly accurate.

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

A Check-In Location Attestation (captured at start time) may contain:
- `expected_branch_location_ref` (reference to the branch’s stored workplace boundary)
- `observed_location` (lat/lng or equivalent location representation)
- `observed_accuracy` (optional; meters or comparable measure)
- `observed_method` (e.g., gps, network, manual — conceptual only)
- `comparison_result` (MATCH, MISMATCH, UNKNOWN)
- `captured_at` (timestamp; usually equal to actual_start_time)
- `notes` (optional; e.g., “permission denied”, “low accuracy”, “not enabled”)

---

## Invariants

Core invariants:
- An attendance record always belongs to exactly one tenant.
- An attendance record is always associated with exactly one staff member.
- An attendance record is always tied to exactly one branch.
- A staff member can have **at most one ACTIVE attendance record at a time**.
- An attendance record always has a start time.
- An attendance record may end later than planned or earlier than planned.
- Attendance describes **reality**, not permission.

Location attestation invariants (only when attestation exists):
- A check-in attestation may be absent (capability disabled or signal unavailable).
- If present, comparison outcome must be one of: **MATCH**, **MISMATCH**, **UNKNOWN**.
- UNKNOWN must be representable (e.g., no permission, no signal, or unreliable accuracy).
- Attestation records evidence and outcome; it does not define punishment or enforcement.

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
- manage payroll or compensation,
- decide which tenants are entitled to location confirmation.

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
- Workplace location boundaries are owned by Branch (or Org configuration), referenced here for comparison.

---

### Attendance ↔ Access Control
- Access Control decides whether work may begin.
- Attendance records work **after** authorization has succeeded.

---

### Attendance ↔ Work Capacity / Licensing
- ACTIVE attendance contributes to concurrent staff capacity.
- COMPLETED attendance frees capacity.

---

### Attendance ↔ Capabilities / Entitlements (Future)
- Whether location attestation is required/collected is controlled by tenant capabilities.
- Attendance remains valid without location attestation.

(Implementation of capabilities is out of scope for March delivery.)

---

### Attendance ↔ Work Review
- Attendance provides raw data for review.
- Interpretation happens elsewhere.

---

## Reality Considerations

- Staff may start work without a planned shift.
- Staff may end work late, early, or unexpectedly.
- Mistakes happen (forgotten check-out).
- GPS/location signals may be inaccurate indoors.
- Permissions may be denied.
- Some tenants may not enable GPS attendance at all.

Manual correction may be required later (handled by separate processes).

The Attendance domain records facts, even when those facts are imperfect.

---

## Out of Scope

- Shift planning
- Authorization logic
- Capacity policy definition
- Payroll and labor law enforcement
- Automated correction or judgment of behavior
- Billing, subscriptions, and entitlement enforcement

---

## Summary

The Attendance domain captures **what actually happened** during work.

It provides:
- a clear boundary for responsibility,
- reliable signals for operational control,
- factual input for later review,
- and optional evidence for check-in location confirmation when enabled.

By separating Attendance from planning, permission, and billing, the system remains fair, auditable, and adaptable.

---

_End of Attendance Domain_
