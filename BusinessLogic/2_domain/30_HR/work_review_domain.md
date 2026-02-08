# Work Review Domain — Interpreting Planned vs Actual Work

## Domain Name
Work Review (Time Respecting & Attendance Insights)

## Domain Type
Interpretation / Reporting-Ready Domain (HR)

## Domain Group
30_HR

## Status
Draft (Derived from Anchor Story A6)

---

## Purpose

The Work Review domain provides **interpreted facts** about work behavior over time by comparing:

- **Shift (planned work expectations)**  
with
- **Attendance (actual work reality)**

Its job is to convert raw records into meaningful, reviewable insights such as:
- lateness,
- early leave,
- absence,
- overtime,
- and consistency over time.

This domain exists so managers and staff can review time respecting fairly, using shared facts rather than memory.

---

## Why This Exists (Story Traceability)

Derived primarily from:

- **Anchor A6 — Reviewing Attendance and Time Respecting**

A6 establishes that:
- review happens after operations,
- patterns matter more than isolated incidents,
- and planned vs actual must be comparable.

Work Review is the domain that makes that comparison explicit.

---

## Core Concepts

### Review Window

A **Review Window** is the time period being reviewed, such as:
- a day,
- a week,
- a month,
- a custom date range.

Review always has a scope.

---

### Work Comparison Result

A **Work Comparison Result** is an interpreted record that answers:
- “How did actual work compare to expectation?”

It is not raw data. It is derived.

Examples of comparison outcomes:
- ON_TIME
- LATE
- EARLY_LEAVE
- ABSENT
- OVERTIME
- UNSCHEDULED_WORK
- INCOMPLETE_RECORD

---

### Review Summary

A **Review Summary** aggregates comparison results into simple metrics:
- number of late arrivals this week,
- average minutes late,
- absence count,
- total overtime minutes,
- reliability indicators (optional, carefully framed).

These are for decision support, not punishment.

---

## Inputs (Owned Elsewhere)

Work Review does not own core truth. It consumes:

From **Shift domain**:
- shift patterns and shift instances (expected work)

From **Attendance domain**:
- attendance records (actual work)
- optional location attestation results (match/mismatch/unknown)

From **Staff Profile**:
- staff identity and branch assignment context

Work Review may also consume:
- branch timezone / business hours (future)

---

## Key Derived Fields (Examples)

A comparison result may include:
- `tenant_id`
- `staff_id`
- `branch_id`
- `review_window_start`
- `review_window_end`
- `expected_start_time` (nullable)
- `expected_end_time` (nullable)
- `actual_start_time` (nullable)
- `actual_end_time` (nullable)
- `classification` (ON_TIME / LATE / ABSENT / etc.)
- `late_minutes` (nullable)
- `early_leave_minutes` (nullable)
- `overtime_minutes` (nullable)
- `evidence_notes` (optional; e.g., “no planned shift”, “no checkout recorded”)

Location-related derived fields (optional):
- `checkin_location_result` (nullable; MATCH / MISMATCH / UNKNOWN)
  - null when location verification was not recorded (mode disabled or missing evidence)

---

## Interpretation Rules (Domain Rules)

Work Review defines interpretation rules that must be consistent.

Examples (conceptual):
- If there is a planned shift but no attendance during that window → ABSENT
- If attendance begins after planned start time beyond tolerance → LATE
- If attendance ends before planned end time beyond tolerance → EARLY_LEAVE
- If attendance exists without a planned shift → UNSCHEDULED_WORK
- If attendance continues beyond planned end time → OVERTIME
- If checkout is missing → INCOMPLETE_RECORD (special case)

Important:
- Tolerances (e.g., 3–5 minutes grace) are configuration/policy inputs.
- Work Review applies the rules, but tolerances are not hard-coded here.

---

## Invariants

- Work Review never edits raw Shift or Attendance history.
- Derived results must be reproducible from source records plus stable rules.
- Interpretation should be explainable (no “black box scoring” required).
- Results must support both fairness and operational usefulness.

Capability-aware invariants:
- Not all tenants will enable advanced review features.
- Core comparison (planned vs actual) should still be possible without paid add-ons.
- Location-based judgments must degrade gracefully if GPS attendance is not enabled.

---

## What This Domain Is NOT

Work Review does **not**:
- enforce attendance rules in real time,
- block check-in or check-out,
- apply punishments,
- manage payroll,
- decide permissions,
- replace human judgment.

It provides interpreted facts and summaries for review.

---

## Relationship to Other Domains

### Work Review ↔ Shift
- Shift provides expectation.
- Work Review interprets adherence to expectation.

### Work Review ↔ Attendance
- Attendance provides reality.
- Work Review interprets reality against expectation.

### Work Review ↔ Access Control
- Access control enforces permissions.
- Work review does not enforce; it reflects.

### Work Review ↔ Reporting
- Work review produces HR-ready metrics that reporting can present.
- Reporting modules may consume review summaries rather than raw records.

---

## Reality Considerations

- Shifts may change frequently.
- Attendance may be imperfect (missing checkout, device issues).
- Managers may override or annotate results later (future enhancement).

The domain should favor **explainability** over complex scoring.

---

## Out of Scope

- Payroll integration
- Performance scoring systems
- Automated disciplinary actions
- AI-based behavior judgment
- Complex labor law compliance

---

## Summary

Work Review is the domain responsible for **interpreting** planned vs actual work and turning raw logs into reviewable insights.

It makes “time respecting” visible through:
- consistent classification,
- explainable metrics,
- and patterns over time.

It supports fairness and business improvement while staying separate from enforcement and billing.

---

_End of Work Review Domain_
