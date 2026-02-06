# Story Coverage Map — Staff Management (People, Branch Access, and Operational Capacity)

**Purpose:**  
Identify human-centered stories for Staff Management behavior and map them to technical use cases, ensuring full coverage without “one story per UC”, while explicitly capturing cross-module stories (authorization, attendance, licensing).

**Scope:**  
- Staff Profile & Assignment Domain (people + branch eligibility)  
- Staff Management Module Spec (v3, simplified onboarding)  
- Cross-cutting: Attendance + concurrent staff capacity

**Rule:**  
Stories are grouped by **human operational context**, not by CRUD actions or internal mechanics.

---

## 1) Source Artifacts (Authoritative)

- **Domain (staff facts):** `BusinessLogic/2_domain/20_HR/staff_profile_and_assignment_domain.md`
- **Domain (attendance + capacity gates):** `BusinessLogic/2_domain/20_HR/attendance_domain_capability_gated.md`
- **ModSpec:** `BusinessLogic/5_modSpec/staffManagement_module_patched_v3.md`

---

## 2) Operational Contexts (Human Reality)

From reading the domain and modspec, staff setup and access is experienced by humans in **five distinct operational contexts**:

1. **Onboarding Staff Quickly (Owner Provisioned)**
2. **Controlling Where People Can Work (Branch Assignment)**
3. **Stopping Access Immediately (Disable / Revoke)**
4. **Keeping Clean History (Archive / Audit)**
5. **Operating Under Fair Limits (Concurrent Staff Capacity)**

These contexts reflect how cafés actually run: shift-based staffing, frequent changes, and strong need for immediate control.

---

## 3) Story Set (Human Situations)

### Context: Onboarding Staff Quickly (Owner Provisioned)

1. **Setting Up the Business Team Quickly**  
   File: `BusinessLogic/1_stories/handling_staff&attendance/setup_staff.md`  
   Human situation: the owner needs to add staff quickly and stay in control (no invite flow).

---

### Context: Controlling Where People Can Work (Branch + Time)

2. **Planning Who Is Expected to Work and Where**  
   File: `BusinessLogic/1_stories/handling_staff&attendance/planning_work_for_staff.md`  
   Human situation: managers plan who should work, when, and at which branch.

---

### Context: Starting and Ending Work (Reality)

3. **Starting Work at a Branch**  
   File: `BusinessLogic/1_stories/handling_staff&attendance/starting_shift.md`  
   Human situation: work begins; the system records the moment and enforces basic boundaries.

4. **Ending Work and Leaving the System Clean**  
   File: `BusinessLogic/1_stories/handling_staff&attendance/ending_shift.md`  
   Human situation: work ends; responsibility and capacity are released.

---

### Context: Operating Under Fair Limits (Guardrails)

5. **Preventing Unauthorized or Excessive Work**  
   File: `BusinessLogic/1_stories/handling_staff&attendance/prevent_unauthorized&excess_work.md`  
   Human situation: the system blocks clearly invalid conditions early (wrong branch, capacity reached).

---

### Context: Optional Location Confirmation (Evidence, Not Surveillance)

6. **Configuring Branch Location for Attendance Confirmation**  
   File: `BusinessLogic/1_stories/handling_staff&attendance/set_branch_gps_location.md`  
   Human situation: define workplace location used to verify check-in/out presence.

---

### Context: Review (Planned vs Actual)

7. **Reviewing Attendance and Time Respecting**  
   File: `BusinessLogic/1_stories/handling_staff&attendance/review_attendance&time_respection.md`  
   Human situation: managers compare planned shifts vs actual attendance over time.

---

**Coverage gaps (not yet explicit as stories):**
- Administrative actions to disable/archive staff (UC-SM2, UC-SM3).
- Reviewing staff change audit trail (staff lifecycle + branch assignment changes).

---

## 4) Coverage Mapping (Use Case → Story)

### Story 1 — Setting Up the Business Team Quickly
Covers:
- UC-SM1 Create Staff Account (Owner Provisioned)
- UC-SM4 Assign Staff to Branch (initial assignment)

Notes:
- Creation does not consume concurrent capacity; concurrency gates operational activity (Attendance / sessions).

---

### Story 2 — Planning Who Is Expected to Work and Where
Covers:
- Branch assignment as an explicit business decision (supports UC-SM4 / UC-SM5 semantics)
- Planned shift expectations (owned by Shift domain, but referenced here)

---

### Story 3 — Starting Work at a Branch
Covers:
- Cross-module: Attendance check-in uses StaffProfile + BranchAssignment facts.
- Cross-module: licensing gates (STAFF_LIMIT_REACHED) are visible at work start.

---

### Story 4 — Preventing Unauthorized or Excessive Work
Covers:
- Failure modes seen by staff at runtime: `NO_BRANCH_ASSIGNMENT`, `STAFF_LIMIT_REACHED`
- Cross-module: deny semantics after disable/unassign (ties to UC-SM2 and UC-SM5 outcomes)

---

### Story 5 — Ending Work and Leaving the System Clean
Covers:
- Cross-module: capacity is freed when operational activity ends (check-out / end work).
- Cross-module: reduced risk of “phantom active staff”.

---

### Story 6 — Configuring Branch Location for Attendance Confirmation
Covers:
- Branch workplace location configuration (used only when location confirmation capability is enabled).

---

### Story 7 — Reviewing Attendance and Time Respecting
Covers:
- Review/reporting: planned shift vs actual attendance evaluation (Work Review domain).

---

### Not Yet Covered As Dedicated Stories
- UC-SM2 Disable Staff (Administrative action)
- UC-SM3 Archive Staff (Administrative action)
- Reviewing staff change audit trail (staff lifecycle + branch assignment changes)

---

## 5) Cross-Module Story Coverage (Do Not Lose These)

These stories are **not owned by Staff Management alone**, but must exist for the product to feel correct.
They should be written under the operational context where users experience them, and referenced by multiple maps.

1. **Starting Work at a Specific Branch**  
   Primary owner: Attendance stories  
   Depends on: Staff Management (branch assignment), Access Control (gate), Staff Licensing (capacity), Authentication (session)

2. **Being Denied Because You Are Not Assigned to This Branch**  
   Primary owner: Access Control stories  
   Depends on: Staff Management (BranchAssignment fact)

3. **Being Denied Immediately After You Were Disabled or Unassigned**  
   Primary owner: Access Control stories  
   Depends on: Staff Management (status/assignment changes) + session behavior

4. **Capacity Block Messaging (What Staff Sees and What Owner Sees)**  
   Primary owner: Staff Licensing stories  
   Depends on: Attendance/Cash Session signals + consistent UX

> Rule: When you write these, keep them human-centered and avoid explaining middleware/authorization internals.

---

## 6) Story File Checklist (Tracking)

| Story | File | Status |
|---|---|---|
| Setting Up the Business Team Quickly | `BusinessLogic/1_stories/handling_staff&attendance/setup_staff.md` | ok |
| Planning Who Is Expected to Work and Where | `BusinessLogic/1_stories/handling_staff&attendance/planning_work_for_staff.md` | ok |
| Starting Work at a Branch | `BusinessLogic/1_stories/handling_staff&attendance/starting_shift.md` | ok |
| Ending Work and Leaving the System Clean | `BusinessLogic/1_stories/handling_staff&attendance/ending_shift.md` | ok |
| Preventing Unauthorized or Excessive Work | `BusinessLogic/1_stories/handling_staff&attendance/prevent_unauthorized&excess_work.md` | ok |
| Configuring Branch Location for Attendance Confirmation | `BusinessLogic/1_stories/handling_staff&attendance/set_branch_gps_location.md` | ok |
| Reviewing Attendance and Time Respecting | `BusinessLogic/1_stories/handling_staff&attendance/review_attendance&time_respection.md` | ok |

Gaps (stories to add later):
- Disable staff (UC-SM2)
- Archive staff (UC-SM3)
- Review staff change audit trail

Legend:
- ⬜ Not written
- 🟨 Drafted
- ✅ Final

---

## 7) Guardrails (Important)

- Do **not** write one story per UC.
- Keep stories **non-technical** and human-centered (no “middleware”, no “auth_account_id”, no “idempotency”).
- “Password rules” are expressed as **trust boundaries**, not cryptography.
- Branch assignment is **explicit** for all roles (including admins).
- Concurrent staff is framed as **fairness to shift-based operations**, not “anti-abuse”.

---

# End of Map
