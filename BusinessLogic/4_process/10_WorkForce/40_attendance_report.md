# 40 — Work Review Reporting

## Purpose

This process exposes evaluated work data so business owners and managers can
understand staff work behavior over time.

This is the final step of the **Work Lifecycle Pipeline**:

1. Work Start
2. Work End
3. Shift vs Attendance Evaluation
4. Work Review Reporting  ← this file

This process turns system data into **human insight**.

---

## Why this process exists

Stores need answers to questions like:

- Who is consistently late?
- Who often leaves early?
- Who frequently misses shifts?
- Are our schedules realistic?
- Which branches struggle with attendance?

These are **business questions**, not technical ones.

This process exists to make the answers visible.

---

## What this process does NOT do

This process does NOT:
- Enforce punishment
- Block staff actions
- Automatically discipline employees

It only provides **visibility and evidence**.

Decisions remain human.

---

## Domains involved

This orchestration connects:

- Work Review Domain → interpreted work outcomes
- Attendance Domain → raw attendance data
- Shift Domain → planned schedules
- Access Control → permission + scope enforcement (branch vs tenant-wide)
- Reporting Module (future consumer)

---

## When this process runs

This is an **on-demand read process**.

Triggered when:
- Managers open attendance dashboards
- Owners review performance reports
- Staff review their own history (future)

This is NOT a background job.

It is a **query and aggregation process**.

---

## Core concept

The system converts many individual work review records into **meaningful summaries**.

Raw data → Patterns → Insights.

---

## Orchestration Steps

### Step 1 — Identify reporting scope

User selects:

- Time range (day/week/month/custom)
- Branch scope (Access Control enforced):
  - a single branch (`branch_id`), OR
  - `ALL_BRANCHES` (tenant-wide; owners/admins only when allowed)
- Staff member (optional)

This defines the **reporting window**.

---

### Step 2 — Retrieve work review records

System fetches Work Review entries that match:

- Selected time range
- Selected branch or tenant
- Selected staff (if applicable)

Work Review entries include:
- expected vs actual timestamps (when available)
- a classification (e.g., ON_TIME / LATE / ABSENT / etc.)
- minute deltas such as `late_minutes`, `early_leave_minutes`, `overtime_minutes` (when applicable)

Domain involved:
- Work Review

---

### Step 3 — Aggregate attendance outcomes

System computes totals such as (derived from Work Review classifications):

Per staff:
- Number of shifts planned
- Number of shifts attended
- Number of absences
- Number of late arrivals
- Number of early departures
- Number of overtime occurrences (optional)
- Number of unscheduled work records (when attendance exists without a planned shift)
- Number of incomplete records (missing checkout or otherwise incomplete evidence)

This creates **behavior indicators**.

Degradation rule:
- If shift planning data is missing for the reviewed window, the system must not present "planned shifts" or "absence" as facts. In that case, show actual attendance-driven summaries and UNSCHEDULED_WORK visibility only.

---

### Step 4 — Calculate time metrics

System calculates:

- Total hours scheduled
- Total hours worked
- Overtime hours
- Missing hours
- Total late minutes (optional)
- Total early leave minutes (optional)

These metrics help evaluate:
- Workforce planning
- Labor utilization

---

### Step 5 — Detect patterns and trends

Over time, patterns become visible:

Examples:
- Frequent late arrivals on Mondays
- High absenteeism at a specific branch
- Staff regularly working overtime

The system surfaces **patterns**, not judgments.

---

### Step 6 — Present insights to users

Insights are presented through:

- Dashboards
- Tables
- Filters and drill-down views

This enables managers to:
- Investigate
- Discuss
- Improve operations

---

## Example outcomes

### Staff attendance summary (monthly)

John:
- Planned shifts: 22
- Attended: 21
- Late arrivals: 3
- Early departures: 1

Manager insight:
John is reliable but occasionally late.

---

### Branch-level insight

Branch A:
- Absence rate: 2%

Branch B:
- Absence rate: 12%

Manager insight:
Branch B may have scheduling or staffing issues.

---

## Relationship to other processes

Consumes output from:
- `30_shift_vs_attendance_evaluation.md`

Provides input to:
- Reporting module (future)
- Management decision-making

---

## Important design principle

This process closes the loop of the **Work Lifecycle**.

The system:
- Plans work
- Records work
- Interprets work
- Reveals insights

It enables better decisions without automating judgment.
