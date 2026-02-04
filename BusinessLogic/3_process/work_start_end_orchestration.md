# Work Start / End Orchestration — Operational Flow

## Process Name
Work Start / End Orchestration

## Process Type
Cross-Domain Orchestration (Identity & HR + POS Operations)

## Status
Draft (Derived from Anchors A3, A4, A5 and Phase 2 Domains)

---

## Purpose

This process defines **how the system coordinates multiple domains** when a staff member starts or ends work.

It ensures that:
- work starts only when allowed,
- work ends cleanly and consistently,
- operational capacity is managed correctly,
- records across domains remain aligned.

This is an **orchestration**, not a domain.  
It does not own business rules; it sequences them.

---

## Why This Exists (Story Traceability)

Derived from:
- **Anchor A3 — Starting Work at a Branch**
- **Anchor A4 — Ending Work and Leaving the System Clean**
- **Anchor A5 — Preventing Unauthorized or Excessive Work**

These stories establish that starting and ending work are **distinct, controlled moments** that require coordination across identity, HR, and operational systems.

---

## Participating Domains

This orchestration coordinates the following domains:

- **Authentication** — who is acting
- **Tenant Membership** — does the person belong to the tenant
- **Staff Profile & Assignment** — who the staff member is and where they may work
- **Access Control** — permission to operate at the branch
- **Attendance** — actual start/end of work
- **Shift (Planned Work)** — expected work (context only)
- **Work Capacity / Licensing** — concurrent staff limits
- **Work Review** — downstream interpretation (not executed here)

No single domain can implement this flow alone.

---

## High-Level Flow Overview

### Work Start (Check-In)
1. Staff authenticates
2. Staff selects tenant and branch
3. System evaluates eligibility and capacity
4. Attendance record is created
5. System confirms work has started

### Work End (Check-Out)
1. Staff initiates end of work
2. Attendance record is completed
3. Capacity is freed
4. System confirms work has ended cleanly

---

## Detailed Orchestration — Work Start

### Step 1: Authentication
- Actor must be authenticated.
- Authentication provides identity reference.
- Failure → stop process.

---

### Step 2: Tenant Membership Validation
- Verify the actor is an ACTIVE member of the tenant.
- Failure → deny start of work.

---

### Step 3: Staff Profile Resolution
- Resolve staff profile linked to the tenant member.
- Verify staff status allows working (e.g., ACTIVE).
- Failure → deny start of work.

---

### Step 4: Branch Context Selection
- Actor selects or operates within a branch.
- Branch must exist and be ACTIVE.
- Failure → deny start of work.

---

### Step 5: Access Control Gate
- Access Control evaluates:
  - action: START_WORK
  - actor identity
  - tenant_id
  - branch_id
- If denied → stop process with explanation.

---

### Step 6: Capacity Gate (Licensing)
- Evaluate concurrent staff capacity.
- ACTIVE attendance count + pending start ≤ allowed capacity.
- If exceeded → deny start of work.

---

### Step 7: Attendance Creation
- Create Attendance Record with:
  - actual_start_time
  - branch_id
  - staff_id
- Attendance status = ACTIVE.

---

### Step 8: Optional Location Confirmation
- If tenant capability allows and signal available:
  - capture location attestation
  - compare with branch workplace location
- Record result as enrichment.
- Location mismatch does **not** automatically fail this process (policy-dependent).

---

### Step 9: Confirmation
- System confirms that work has started.
- Staff is now considered ACTIVE for:
  - capacity
  - responsibility
  - operational tracking

---

## Detailed Orchestration — Work End

### Step 1: Identify Active Attendance
- Resolve ACTIVE attendance record for the staff member.
- If none exists → no-op or error (policy).

---

### Step 2: Attendance Completion
- Set actual_end_time.
- Attendance status = COMPLETED.
- Attendance history is preserved.

---

### Step 3: Capacity Release
- Completed attendance frees one unit of concurrent capacity.
- No further checks required.

---

### Step 4: Confirmation
- System confirms that work has ended.
- Staff is no longer considered operationally active.

---

## Failure Modes (Conceptual)

### During Work Start
- Not authenticated
- Not a tenant member
- Staff inactive or archived
- Branch inactive
- Access denied
- Capacity exceeded

Each failure must return a **clear, human-readable reason**.

---

### During Work End
- No active attendance
- Duplicate end request
- System interruption

The system should favor **idempotent and safe completion**.

---

## What This Process Is NOT

This orchestration does **not**:
- plan shifts
- enforce lateness rules
- discipline staff
- calculate payroll
- manage billing or subscriptions

It only ensures **clean transitions into and out of active work**.

---

## Relationship to Other Processes

- Feeds **Work Review** via Attendance records
- Feeds **Reporting** via derived summaries
- May trigger **Operational Notifications** (future)
- May interact with **Cash Session** (POS context)

---

## Summary

Work Start / End Orchestration defines how the system safely and consistently handles the **moment someone begins or stops working**.

By sequencing authentication, membership, access control, capacity checks, and attendance recording, it ensures:
- clarity,
- fairness,
- and operational safety,

even under real-world pressure.

---

_End of Work Start / End Orchestration_
