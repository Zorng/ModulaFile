# 30 — Shift vs Attendance Evaluation

## Purpose

This process compares **planned work (Shift)** with **actual work (Attendance)** to produce meaningful work insights.

It is the bridge between:
- What was *expected*
- What actually *happened*

This orchestration feeds the **Work Review domain**.

---

## Why this process exists

Planning alone is not enough.  
Attendance alone is not enough.

A store needs to answer questions like:

- Did staff arrive on time?
- Did they leave early?
- Did they skip work?
- Did they work outside their shift?
- Is the schedule realistic?

These answers cannot come from a single domain.

They emerge from comparing:
"Shift (Plan)  vs  Attendance (Reality)"

---

## When this process runs

This is a **background evaluation process**.

Typical triggers:

- After a shift window ends
- Periodic daily evaluation job
- On-demand evaluation request (future)

This is **not a real-time blocking process**.

It is an **interpretation process**.

---

## Domains involved

This orchestration connects:

- Shift Domain → planned work
- Attendance Domain → actual work
- Work Review Domain → interpretation

---

## High-level outcome

For each planned shift, the system determines:

- Attendance status
- Time respecting status
- Exceptions and anomalies

The output becomes **Work Review records**.

---

## Core concept

We compare **expected timeline** with **actual timeline**.

Example:

| Type       | Start | End   |
| ---------- | ----- | ----- |
| Shift      | 08:00 | 16:00 |
| Attendance | 08:12 | 15:40 |

The system interprets:
- Late arrival
- Early departure
- Shorter work duration

---

## Orchestration Steps

### Step 1 — Identify evaluation window

System selects shifts that:

- Have already ended
- Have not yet been evaluated

This ensures:
- No duplicate evaluations
- Deterministic processing

Domain involved:
- Shift

---

### Step 2 — Find matching attendance

System searches attendance records for:

- Same staff
- Same branch
- Same date
- Overlapping time window

Possible outcomes:

1) Matching attendance found  
2) No attendance found

Domains involved:
- Attendance

---

### Step 3 — Determine attendance status

If **no attendance found**:
- Status = `Absent`

If attendance exists:
- Continue evaluation.

---

### Step 4 — Evaluate arrival behavior

Compare:
`Attendance.check_in vs Shift.start_time`

Possible outcomes:

- On time
- Late arrival
- Early arrival

Tolerance rules may apply (future policy).

---

### Step 5 — Evaluate departure behavior

Compare:
`Attendance.check_out vs Shift.end_time`

Possible outcomes:

- On time
- Early departure
- Overtime work

---

### Step 6 — Evaluate total work duration

Compare:
`Actual worked duration vs Planned shift duration`

This detects:
- Under-worked shifts
- Over-worked shifts

---

### Step 7 — Detect anomalies

Examples of anomalies:

- Attendance without shift
- Shift without attendance
- Multiple check-ins
- Extremely short sessions
- Extremely long sessions

These are recorded as **exceptions**.

---

### Step 8 — Create Work Review record

Work Review domain receives:

- Attendance status
- Time respecting indicators
- Duration comparison
- Detected anomalies

This becomes the foundation for:
- Reports
- Manager insights
- Future automation

Domain involved:
- Work Review

---

## Important design principle

This process does **not punish** or **enforce policy**.

It only:
- Observes
- Compares
- Records

Interpretation belongs to humans.

---

## Example outcomes

### Perfect shift

Shift: 08:00–16:00  
Attendance: 07:58–16:03  

Result:
- On time arrival
- On time departure
- No anomalies

---

### Late arrival

Shift: 08:00–16:00  
Attendance: 08:21–16:00  

Result:
- Late arrival detected

---

### Absent

Shift: 08:00–16:00  
Attendance: none  

Result:
- Absent

---

### Unexpected work

Shift: none  
Attendance: 09:00–13:00  

Result:
- Attendance without planned shift (exception)

---

## Relationship to other processes

Depends on:
- `10_work_start_end_orchestration.md`
- `20_work_end_orchestrastion.md`

Feeds into:
- Work Review domain
- Reporting (future)

This completes the **Work Lifecycle pipeline**:
1. Start Work
2. End Work
3. Evaluate Work
