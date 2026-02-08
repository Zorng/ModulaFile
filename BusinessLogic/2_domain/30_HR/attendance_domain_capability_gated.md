# Attendance Domain — Actual Work

## Domain Name
Attendance (Actual Work)

## Domain Type
HR / Operational Reality Domain

## Domain Group
30_HR

## Status
Draft — rewritten (supports dual location verification + capability gating)

---

## Purpose

The Attendance domain represents **what actually happened during work**.

Attendance records the moments when a staff member **starts** and **ends** working at a branch.  
It captures reality as it occurred, regardless of what was planned.

This domain exists to:

- establish a reliable source of truth for actual work
- support operational control during the day
- support concurrent staff capacity enforcement (Attendance is the ACTIVE-work signal; capacity policies are owned elsewhere)
- enable later review of time respecting and behaviour

Attendance is about **events that happened**, not what should have happened.

---

## Why This Exists (Story Traceability)

Derived from anchor stories:

- Starting Work at a Branch  
- Ending Work and Leaving the System Clean  
- Reviewing Attendance and Time Respecting  

Without Attendance, the system cannot know:

- who is currently working
- when responsibility started or ended
- whether capacity limits are respected
- what actually happened over time

---

## Key Design Philosophy

Attendance provides **operational evidence**, not surveillance.

Modula intentionally avoids continuous tracking.  
Instead, the system captures **verification points**.

This approach balances:

- privacy
- battery usage
- reliability
- product positioning (POS ≠ surveillance tool)

This model is commonly used in modern retail systems and is called:

**Point Verification Attendance**

---

## Core Concept

### Attendance Record

An **Attendance Record** represents a continuous period of real work by a staff member at a branch.

It answers:

- Who worked?
- Where did they work?
- When did they start?
- When did they finish?

An attendance record may or may not correspond to a planned shift.

---

## Dual Location Verification (Optional Capability)

Attendance may include **location verification at two moments**:

1. Check-in verification  
2. Check-out verification  

These checkpoints provide reasonable confidence that work happened at the workplace.

The system **does not attempt continuous tracking**.

---

## Why Dual Verification Exists

Without checkout verification, the system cannot distinguish between:

- staff who worked until the end of shift
- staff who left early but returned to tap “checkout”

Dual verification provides **start and end anchors** without invasive tracking.

This is the recommended configuration for Modula POS.

---

## Capability Gating

Attendance location verification is controlled by tenant capability:
`attendance.location_verification_mode`

Possible values:

| Mode                 | Meaning                                  |
| -------------------- | ---------------------------------------- |
| disabled             | No GPS verification                      |
| checkin_only         | Verify location at start of work         |
| checkin_and_checkout | Verify location at start and end of work |

This supports Modula’s philosophy:

> You only pay for what you use.

Attendance must function correctly in all three modes.

---

## Attendance Confidence Levels

### Level 1 — Unverified Attendance
Evidence:
- check-in time
- check-out time

Used by small or trust-based teams.

---

### Level 2 — Start-Verified Attendance
Evidence:
- verified check-in location
- declared check-out time

Provides moderate confidence.

---

### Level 3 — Start & End Verified Attendance (Recommended)
Evidence:
- verified check-in location
- verified check-out location
- full work duration

Provides strong operational confidence.

---

## Location Verification Model

Location verification records **evidence**, not judgment.

Verification captures:

- observed device location
- comparison with branch workplace boundary
- accuracy and method (optional metadata)
- comparison result: MATCH / MISMATCH / UNKNOWN

Important:
- Location verification is **evidence-first**. It does not block check-in or check-out.
- The result is used for visibility and later review by managers/admins.

UNKNOWN must always be possible:
- permission denied
- weak signal
- indoor GPS drift
- branch workplace location not configured

If location verification mode is `disabled`, verification fields are simply not recorded (not UNKNOWN).

The system records evidence without assuming perfect accuracy.

---

## Key Attributes

### Attendance Record

- attendance_id
- tenant_id
- staff_id
- branch_id
- actual_start_time
- actual_end_time (nullable while active)
- status (ACTIVE | COMPLETED)
- created_at
- updated_at

### Check-In Verification (optional)

- checkin_observed_location
- checkin_accuracy
- checkin_method
- checkin_result (MATCH / MISMATCH / UNKNOWN)

### Check-Out Verification (optional)

- checkout_observed_location
- checkout_accuracy
- checkout_method
- checkout_result (MATCH / MISMATCH / UNKNOWN)

---

## Core Invariants

- Attendance belongs to exactly one tenant.
- Attendance belongs to exactly one staff member.
- Attendance occurs at exactly one branch.
- A staff member may have **at most one ACTIVE attendance record**.
- Attendance always has a start time.
- Attendance describes **reality**, not permission.
- Attendance records are never deleted.

---

## Attendance Lifecycle

States:

- ACTIVE — work has started
- COMPLETED — work has ended

Attendance is historical fact and must remain immutable after completion.

---

## What Attendance Does NOT Do

Attendance does **not**:

- decide who is allowed to start work (Access Control)
- plan schedules (Shift domain)
- define staff capacity limits (Licensing / Entitlements domain), though Attendance consumes those limits at check-in/out as an enforcement point
- interpret performance (Work Review domain)
- handle payroll or compensation
- enforce subscriptions or billing

Attendance records events only.

---

## Relationship to Other Domains

### Attendance ↔ Shift
Shift = expectation  
Attendance = reality  

They are compared later.

---

### Attendance ↔ Access Control
Authorization happens **before** attendance begins.

---

### Attendance ↔ Capacity / Licensing
ACTIVE attendance consumes staff capacity.  
COMPLETED attendance frees capacity.

---

### Attendance ↔ Branch
Attendance references the branch workplace and its location boundary.

---

### Attendance ↔ Work Review
Attendance provides raw data for evaluation.  
Interpretation happens in another domain.

---

## Real-World Considerations

- Staff may start work without a planned shift.
- Staff may forget to check out.
- GPS may be inaccurate indoors.
- Permissions may be denied.
- Some tenants may disable GPS entirely.

Attendance must remain reliable even when signals are imperfect.

---

## Summary

The Attendance domain records **what actually happened during work**.

It provides:

- operational awareness
- capacity control
- factual historical records
- optional location verification evidence

It intentionally avoids surveillance while providing strong operational signals.

---

_End of Attendance Domain_
