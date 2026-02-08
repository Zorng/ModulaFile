# attendance_module.md

**Version:** 1.4  
**Status:** Patched (Shift schedules added in Staff Management; attendance shift evaluation staged)  
**Module Type:** Feature Module  
**Depends on:** Authentication & Authorization (Core), Tenant & Branch Context (Core), Policy & Configuration (Core), Sync & Offline Support (Core), Audit Logging (Core)  
**Related Modules:** Staff Management, Cash Session, Branch (Core/Context)

---

## Module: Attendance (Feature Module)

---

## Purpose

The Attendance module records and manages staff presence at branches by capturing check-in and check-out events. It enforces attendance-related policies, supports offline-first operation, and provides visibility to managers and administrators without allowing mutation of recorded attendance data.

This module is designed for small to medium F&B businesses where attendance must be simple to operate yet auditable and policy-driven.

---

## Scope (Capstone I)

Included:
- Staff check-in and check-out
- Attendance always recorded in **branch context**
- **Branch-scoped** policy enforcement for attendance behavior
- Out-of-shift check-in approval workflow
- Role-based access to attendance history
- Offline-first attendance recording with later synchronization

Excluded (Capstone II+):
- Automated reminders / notifications
- Auto check-out
- Attendance analytics and anomaly detection
- Editing or deleting attendance records

Location verification is an **optional** capability:
- check-in only, or
- check-in and check-out

Design rule (locked): location verification is **evidence-first** and must not block check-in/out. The system records `MATCH/MISMATCH/UNKNOWN` and lets managers/admins interpret the flags.

---

## Key Concepts

- **Attendance Record**: An immutable record representing a staff check-in or check-out.
- **Check-in Request**: A temporary state created when a staff attempts to check in outside allowed policy (e.g., out of shift) and requires approval.
- **Branch Context (System-owned)**:
  - Staff do **not** create or select branches.
  - A branch exists because the **system created it** (e.g., tenant provisioning + subscription add-branch in future scope).
  - A staff member’s effective branch is determined by their membership/assignment in Staff Management / Tenant & Branch Context.
- **Branch Status**:
  - If a branch is **Frozen/Inactive**, attendance actions must be blocked for that branch.
- **Policy Enforcement (Branch-scoped)**:
  - Attendance behavior is controlled by policies defined in the Policy & Configuration module **per branch**.
  - The effective attendance policy is resolved by `(tenant_id, branch_id)`.
  - Different branches under the same tenant may have different policy values.

- **Shift Schedule (Branch-scoped)**:
  - Weekly staff shift schedules are assigned by Admin via Staff Management (per staff + branch).
  - Attendance uses shift schedules to determine early / out-of-shift behavior.

**Implementation note (Capstone 1, current backend):**
- Shift evaluation is staged. The attendance check-in API currently accepts a client-provided `shiftStatus` (`IN_SHIFT | EARLY | OUT_OF_SHIFT`) and optional `earlyMinutes` as an interim input.
- Future work: backend computes shiftStatus from stored shift schedules + policy buffer rules.

---

## Actors

- **Cashier**
- **Manager**
- **Admin**
- **System**

---

## Use Cases

### UC-1: Staff Check-in (Cashier / Manager)

**Actors**: Cashier, Manager  
**Preconditions**:
- User is authenticated
- User is assigned to a branch (branch_id resolved from membership/assignment)
- Assigned branch is ACTIVE (not frozen)
- User is not currently checked in
- If offline: device has cached branch assignment (last known) and policies (last known)

**Main Flow**:
1. User initiates check-in from their portal.
2. System resolves `branch_id` from the user’s assignment (no branch selection UI).
3. System resolves effective **branch-scoped** attendance policies for `(tenant_id, branch_id)`.
4. System evaluates policy constraints:
   - Early check-in buffer
   - Shift assignment
   - Out-of-shift approval requirement
5. If policy allows:
   - Attendance record is created (CHECK_IN).
6. If out-of-shift and approval is required:
   - A check-in request is created with status `PENDING_APPROVAL`.

**Postconditions**:
- User is either checked in (active attendance) or has a pending approval request.

**Acceptance Criteria**:
- User cannot check in to a different branch.
- System prevents duplicate active check-ins.
- If branch is frozen/inactive, check-in is blocked with a clear message.
- Offline check-ins are stored locally and synced later.
- Policy evaluation uses the branch the user is assigned to at the time of check-in.

---

### UC-2: Staff Check-out (Cashier / Manager)

**Actors**: Cashier, Manager  
**Preconditions**:
- User has an active check-in
- Assigned branch is ACTIVE (not frozen)  
  (If branch becomes frozen after check-in, check-out is still allowed to close attendance cleanly.)

**Main Flow**:
1. User initiates check-out.
2. System records check-out timestamp (CHECK_OUT).

**Postconditions**:
- Attendance session is closed.

**Acceptance Criteria**:
- Late check-out is allowed.
- No automatic or forced check-out occurs.
- Timestamps are immutable after creation.

---

### UC-3: View Own Attendance History

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

### UC-4: View Branch Attendance History

**Actors**: Manager  
**Preconditions**:
- Manager is assigned to a branch

**Main Flow**:
1. Manager accesses branch attendance view.
2. System displays attendance records for staff in that manager’s assigned branch.

**Postconditions**:
- None (read-only)

**Acceptance Criteria**:
- Manager cannot view other branches.
- Manager cannot edit or delete records.

---

### UC-5: View All Attendance History

**Actors**: Admin  
**Preconditions**:
- Admin is authenticated

**Main Flow**:
1. Admin accesses global attendance view.
2. System displays attendance records across all branches.

**Postconditions**:
- None (read-only)

**Acceptance Criteria**:
- Admin cannot edit timestamps or delete records.
- Admin can filter by branch, staff, or date.

---

### UC-6: Approve Out-of-Shift Check-in

**Actors**: Manager, Admin  
**Preconditions**:
- A check-in request exists with status `PENDING_APPROVAL`
- Request belongs to a valid branch
- Approver has authority for the request’s branch:
  - Manager: own branch only
  - Admin: any branch
- Branch is ACTIVE (not frozen)

**Main Flow**:
1. Manager or Admin reviews the pending request.
2. Actor approves or rejects the request.
3. If approved:
   - System creates or activates the attendance record (CHECK_IN).
4. If rejected:
   - Request is marked rejected and no attendance is created.

**Postconditions**:
- Check-in request is resolved.

**Acceptance Criteria**:
- Approval does not modify timestamps.
- Approval action is logged in Audit Log.
- Rejected requests do not create attendance records.

---

## Policies Used (Read-only from Policy Module)

**Scope:** Attendance policies are resolved per `(tenant_id, branch_id)`.

- `attendanceEarlyCheckinBufferEnabled` (on/off)
- `attendanceCheckinBufferMinutes` (duration)
- `attendanceRequireOutOfShiftApproval` (on/off)

Notes:
- If a policy value is missing for a branch, the system applies default values (as defined by Policy module defaults).
- Attendance module does not create policies; it only reads effective policy values.

---

## Data Integrity Rules

- Attendance records are immutable once created.
- Approval actions do not modify recorded timestamps.
- Historical attendance remains tied to the original branch context at time of record creation.
- Offline-created records are reconciled during sync with policy enforcement (branch-scoped).

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
- Admin visibility is global, but authority to mutate attendance data is intentionally restricted.

---

## Audit Events Emitted

The following events MUST be written to the Audit Log when triggered (with tenant_id, branch_id, actor_id, and relevant entity IDs):

- `ATTENDANCE_CHECKIN_CREATED`
- `ATTENDANCE_CHECKOUT_CREATED`
- `ATTENDANCE_OUT_OF_SHIFT_REQUESTED`
- `ATTENDANCE_OUT_OF_SHIFT_APPROVED`
- `ATTENDANCE_OUT_OF_SHIFT_REJECTED`
