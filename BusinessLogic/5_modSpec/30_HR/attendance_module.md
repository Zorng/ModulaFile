# attendance_module.md

**Version:** 1.5  
**Status:** Patched (Aligned to Work Start/End orchestration; evidence-first location verification; shift alignment is soft)  
**Module Type:** Feature Module  
**Depends on:** Authentication, Access Control, Tenant (membership facts), Staff Management (assignment facts), Branch, Shift, Policy (optional), Sync & Offline, Audit Logging  
**Related Modules:** Work Review, Cash Session (boundary), Sale (boundary), Reporting (future)

---

## Module: Attendance (Feature Module)

---

## Purpose

The Attendance module records and manages staff presence at branches by capturing check-in and check-out events. It supports offline-first operation and provides visibility to managers and administrators without allowing mutation of recorded attendance data.

This module is designed for small to medium F&B businesses where attendance must be simple to operate yet auditable and policy-driven.

---

## Scope (Capstone I)

Included:
- Start work (check-in) and end work (check-out)
- Attendance is always recorded in **tenant + branch context**
- One ACTIVE work session per staff (per tenant); idempotent under retries/offline replay
- Optional location verification evidence (capability-gated; never blocks check-in/out)
- Role-scoped read access to attendance history
- Manager/Admin forced checkout (correction) with audit logging
- Offline-first capture with later synchronization

Excluded (Capstone II+):
- Automated reminders / notifications
- Auto check-out
- Attendance analytics and anomaly detection
- Editing or deleting attendance records
- Out-of-shift approval workflows (deferred; handled as review signals instead of real-time enforcement)

Location verification is an **optional** capability (point verification attendance):
- check-in only (`checkin_only`)
- check-in and check-out (`checkin_and_checkout`)

Design rule (locked): location verification is **evidence-first** and must not block check-in/out. The system records `MATCH/MISMATCH/UNKNOWN` and lets managers/admins interpret the flags.

---

## Key Concepts

- **Attendance Record (Work Session)**: An immutable record representing a continuous period of actual work (ACTIVE → COMPLETED).
- **Branch Context (System-created, user-selected when needed)**:
  - Branch records are system-provisioned (not created by staff).
  - A staff member may be assigned to one or multiple branches.
  - If multiple branch assignments exist, the user must select a branch context before starting work (validated by Access Control).
  - Branch switching mid-shift is not allowed: END_WORK at Branch A before START_WORK at Branch B.
- **Branch Status**:
  - If a branch is **FROZEN** (not ACTIVE), start-work/check-in must be blocked for that branch.
  - March baseline: end-work/check-out is also blocked when branch is frozen (no safe-closure exception is specified).
- **Shift Alignment (Soft)**:
  - Shift schedules represent planned expectations (Shift domain).
  - Attendance does not hard-block out-of-shift work; it records timestamps and (optionally) shift alignment context for later Work Review.
- **Location Verification Evidence (Capability-gated)**:
  - When enabled, the system captures device location at check-in and/or check-out and records:
    - observed location + accuracy (if available)
    - evaluation result: `MATCH` / `MISMATCH` / `UNKNOWN`
  - UNKNOWN is valid when permission is denied, signal is weak, or branch workplace location is not configured.

**Authoritative processes and contracts**
- Work start orchestration: `BusinessLogic/4_process/10_WorkForce/10_work_start_end_orchestration.md`
- Work end orchestration: `BusinessLogic/4_process/10_WorkForce/20_work_end_orchestrastion.md`
- Shift vs attendance evaluation: `BusinessLogic/4_process/10_WorkForce/30_shift_vs_attendance_evaluation.md`
- Work review reporting: `BusinessLogic/4_process/10_WorkForce/40_attendance_report.md`
- HR edge cases: `BusinessLogic/3_contract/10_edgecases/identity_hr_edge_case_sweep_patched.md`
- HR ↔ POS boundary: `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md`

---

## Subscription & Entitlements Integration (Billing Guard Rails)

Attendance is a Workforce capability and is branch-scoped for enforcement.

- **Entitlement key (branch-scoped):** `module.workforce`
- If `module.workforce` enforcement is `READ_ONLY` (not subscribed):
  - allow viewing attendance history (read actions),
  - block attendance start/end writes (`START_WORK`, `END_WORK`).
- If tenant subscription state is `FROZEN`:
  - operational writes are blocked (Attendance becomes view-only).

Enforcement is performed by Access Control using action metadata (scope + effect) and entitlements.

---

## Actors

- **Cashier**
- **Manager**
- **Admin**
- **System**

---

## Use Cases

### UC-1: Start Work (Check-in)

**Actors**: Cashier, Manager, (Admin if operational)  
**Preconditions**:
- User is authenticated (Authentication)
- Tenant context is selected
- Branch context is selected/resolved (required if multiple assignments exist)
- Access Control authorizes START_WORK in `{tenant_id, branch_id}`
- Staff has ACTIVE membership + ACTIVE staff profile + ACTIVE branch assignment for the branch
- Branch is ACTIVE (not FROZEN)
- Staff has no existing ACTIVE attendance session in this tenant
- Operator seat capacity is available (Subscription & Entitlements gate via orchestration)

**Main Flow**:
1. User taps **Start Work / Check In**.
2. System executes Work Start orchestration:
   - `BusinessLogic/4_process/10_WorkForce/10_work_start_end_orchestration.md`
3. (Soft) Shift alignment is evaluated if shift planning exists and recorded as context (not a hard gate).
4. (Capability) If `attendance.location_verification_mode` is `checkin_only` or `checkin_and_checkout`, capture and evaluate location:
   - record `MATCH/MISMATCH/UNKNOWN`
   - do not block check-in on GPS issues
5. Attendance record is created as ACTIVE (check-in time + branch + evidence/context).

**Postconditions**:
- Staff has one ACTIVE attendance session for the tenant (in the selected branch).
- Operator seat is consumed (concurrent operator seats).

**Acceptance Criteria**:
- Duplicate check-in is idempotent (returns the existing ACTIVE session).
- If branch is FROZEN, check-in is blocked with a clear reason.
- If GPS permission is denied / signal weak / branch workplace location missing, check-in succeeds and evidence result is UNKNOWN.
- If staff is assigned to multiple branches, the branch must be selected before check-in.

---

### UC-2: End Work (Check-out)

**Actors**: Cashier, Manager, (Admin if operational)  
**Preconditions**:
- User has an ACTIVE attendance session in the current tenant + branch context
- Work End orchestration confirms there are no unresolved responsibilities (e.g., open cash session)
- If branch becomes frozen after check-in, check-out may be blocked (March baseline has no safe-closure exception specified)

**Main Flow**:
1. User taps **End Work / Check Out**.
2. System executes Work End orchestration:
   - `BusinessLogic/4_process/10_WorkForce/20_work_end_orchestrastion.md`
3. System blocks end-work if responsibilities remain open (see boundary contract):
   - `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md`
4. (Capability) If `attendance.location_verification_mode` is `checkin_and_checkout`, capture and evaluate checkout location evidence (never blocks).
5. Attendance session is closed (ACTIVE → COMPLETED) with check-out time; capacity slot is released.

**Postconditions**:
- Attendance session is closed.

**Acceptance Criteria**:
- Late check-out is allowed.
- Duplicate end-work is idempotent (returns success without double-closing).
- Checkout is blocked if cash session (or other responsibilities) are still open.
- Timestamps are immutable after creation.

---

### UC-3: Force End Work (Manager/Admin)

**Actors**: Manager, Admin  
**Preconditions**:
- A target staff member has an ACTIVE attendance session
- Approver is authorized for the tenant/branch context
- Forced end-work must not erase open responsibilities (e.g., cash session must be resolved first)

**Main Flow**:
1. Manager/Admin selects a staff member → **Force Check Out** and provides a reason.
2. System executes Work End orchestration with `forced_by` metadata.
3. Attendance session closes (ACTIVE → COMPLETED); capacity slot is released.
4. Audit log records who forced the checkout and why.

**Postconditions**:
- Staff is no longer ACTIVE working.

---

### UC-4: View Own Attendance History

**Actors**: Cashier, Manager  
**Preconditions**:
- User is authenticated

**Main Flow**:
1. User navigates to attendance history.
2. System displays only the user’s own records.

**Postconditions**:
- None (read-only)

**Acceptance Criteria**:
- User cannot view other staff’s attendance.
- Records are sorted by date/time.

---

### UC-5: View Branch Attendance History

**Actors**: Manager  
**Preconditions**:
- Manager is authorized for the selected branch context

**Main Flow**:
1. Manager accesses branch attendance view.
2. System displays attendance records for staff in that branch.

**Postconditions**:
- None (read-only)

**Acceptance Criteria**:
- Manager cannot view other branches.
- Manager cannot edit or delete records.

---

### UC-6: View Tenant Attendance History

**Actors**: Admin  
**Preconditions**:
- Admin is authenticated
- Admin is authorized for tenant-level attendance visibility

**Main Flow**:
1. Admin accesses global attendance view.
2. System displays attendance records across all branches.

**Postconditions**:
- None (read-only)

**Acceptance Criteria**:
- Admin cannot edit timestamps or delete records.
- Admin can filter by branch, staff, or date.

---

## Capability & Policy Inputs

### Capability: Location Verification Mode (Tenant-level)
`attendance.location_verification_mode`
- `disabled`
- `checkin_only`
- `checkin_and_checkout`

### Branch Workplace Location (Branch-level configuration)
Used only when location verification is enabled:
- workplace latitude/longitude
- allowed radius (meters)

If workplace location is missing, verification result must be recorded as UNKNOWN (do not block check-in/out).

### Policy note (avoid drift)
Some legacy policy keys exist in the current policy schema (e.g., out-of-shift approval), but the March baseline Work Start/End orchestrations treat shift alignment as a **soft** signal for Work Review, not a real-time approval workflow.

---

## Data Integrity Rules

- Attendance records are immutable once created.
- An attendance session may be closed (end time appended), but historical start time must never be edited.
- A staff member may have at most one ACTIVE attendance session per tenant.
- Historical attendance remains tied to the original branch context at time of record creation.
- Offline-created records must be idempotent under sync replay (use client operation IDs).
- Attendance start/end writes must pass the platform idempotency gate:
  - `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`

---

## Non-Functional Requirements

- Offline-first support using local storage and sync queue
- Idempotent synchronization
- Role-based access control enforced server-side
- All actions logged via Audit Logging module

---

## Notes

- Starting or closing a cash session is **not** equivalent to check-in or check-out.
- Cash Session and Attendance modules are independent. Policy may reference both, but data integrity remains separated.
- Work end must not erase open responsibilities (e.g., cash session). This rule is owned by Work End orchestration + boundary contract.

---

## Audit Events Emitted

The following events MUST be written to the Audit Log when triggered (with tenant_id, branch_id, actor_id, and relevant entity IDs):

- `CHECK_IN_CREATED`
- `CHECK_OUT_CREATED`
- `CHECK_OUT_FORCED` (if force checkout is implemented as a distinct event; otherwise enrich `CHECK_OUT_CREATED` with forced metadata)
