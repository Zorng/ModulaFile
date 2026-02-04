# staffManagement_module.md

## Module: Staff Management (Feature Module)

**Version:** 1.2  
**Status:** Patched (Adds staff shift scheduling; aligned with Branch creation model + Capstone 1 boundaries)  
**Module Type:** Feature Module  
**Depends on:** Auth & Authorization (Core), Tenant & Branch Context (Core), Policy & Configuration (Core), Audit Logging (Core)  
**Related Modules:** Staff Attendance, Cash Session, Reporting, Branch (Core/Context)

---

## 1. Purpose

The Staff Management module manages **workforce membership within a tenant**:
- Who can access a tenant’s POS
- What **role** they have (Admin / Manager / Cashier)
- Which **branch** they are assigned to (when applicable)

This module focuses on **organizational access**, not personal identity or credentials.

Personal account data (password, OTP verification, phone ownership, account recovery, display name changes) belongs to **Authentication & Account Settings**, not this module.

---

## 2. Scope (Capstone I)

### Included
- Inviting staff into a tenant
- Assigning branch and role (from existing branches)
- Assigning weekly shift schedules to staff (per branch)
- Activating, disabling, archiving, and reactivating staff memberships
- Enforcing staff seat limits (soft & hard limits)
- Viewing staff list and status
- Audit logging of staff lifecycle changes

### Explicitly Not Included
- Creating branches (branches are system-created)
- Password reset / OTP flows
- Phone verification
- Editing personal account details (name, credentials)
- Payroll or HR data
- Attendance mutation (handled by Attendance module)

---

## 3. Key Clarification: Branches Are System-Created (Not User-Created)

Branches do **not** originate from Staff Management.

Branch lifecycle is created/expanded by the **system**, driven by:
- Initial tenant provisioning (system creates the first branch when tenant is created)
- Subscription upgrades that add branch capacity (future billing engine; currently developer-admin flow)

Staff Management can only:
- **Select** from existing branches that belong to the tenant
- Assign staff to those branches
- Respect branch state (e.g., frozen/disabled branches cannot receive new assignments)

---

## 4. Core Concepts

### 4.1 Staff Membership
A record linking a **User Account** to a **Tenant**, defining:
- Role (Admin / Manager / Cashier)
- Branch assignment (where applicable)
- Membership status

### 4.2 Invitation
A temporary record created by Admin to onboard staff into a tenant.

### 4.3 Staff Status
- **INVITED**: Invitation sent, not yet accepted
- **ACTIVE**: Staff can access POS
- **DISABLED**: Temporarily blocked from access (does not consume an active slot)
- **ARCHIVED**: Permanently removed from active use (counts toward hard limit)

---

## 5. Actors

- **Admin (Tenant)**
- **Manager (Branch)**
- **System** (automation and cross-module enforcement)

---

## 6. Use Cases

### UC-1: View Staff List

**Actors:** Admin, Manager

#### Preconditions
- Actor is authenticated
- Actor has access to tenant context

#### Main Flow
1. Actor opens Staff Management screen.
2. System displays staff list with:
   - Identifier (e.g., phone)
   - Role
   - Branch assignment
   - Status (Invited / Active / Disabled / Archived)

#### Postconditions
- None (read-only)

#### Acceptance Criteria
- Admin sees staff across all branches.
- Manager sees staff only in their assigned branch.
- Cashier cannot view staff list.

---

### UC-2: Invite Staff

**Actors:** Admin

#### Preconditions
- Admin is authenticated
- Tenant has not exceeded **hard staff limit**
- Selected branch (if required) **exists** and is **not frozen**

#### Main Flow
1. Admin selects “Invite Staff”.
2. Admin inputs:
   - Phone number
   - Role
   - Branch assignment (chosen from tenant’s existing branches)
3. System creates an invitation.
4. Invitation is sent to staff for onboarding.

#### Postconditions
- Staff membership created with status `INVITED`.

#### Acceptance Criteria
- Cannot invite if hard limit reached.
- Cannot assign to a frozen branch.
- Invitation can expire or be revoked.
- Action is written to the audit log.

---

### UC-3: Accept Invitation (Onboarding)

**Actors:** System (triggered by staff)

#### Preconditions
- Valid invitation exists
- Invitation not expired or revoked
- Tenant has available **active staff slots** (soft limit)

#### Main Flow
1. Staff completes authentication via Auth module.
2. System activates staff membership.

#### Postconditions
- Staff status becomes `ACTIVE`.

#### Acceptance Criteria
- Staff is assigned correct role and branch.
- Activation fails if soft limit is exceeded.

---

### UC-4: Disable Staff

**Actors:** Admin

#### Preconditions
- Staff status is `ACTIVE`

#### Main Flow
1. Admin selects staff member.
2. Admin chooses “Disable”.
3. System updates status to `DISABLED`.

#### Postconditions
- Staff access to POS is blocked.

#### Acceptance Criteria
- Disabled staff does **not** count toward soft (active) slots.
- Historical attendance and sales remain unchanged.
- Action is written to the audit log.

---

### UC-5: Reactivate Staff

**Actors:** Admin

#### Preconditions
- Staff status is `DISABLED`
- Tenant has available active staff slots (soft limit)

#### Main Flow
1. Admin selects staff member.
2. Admin chooses “Reactivate”.
3. System sets status to `ACTIVE`.

#### Postconditions
- Staff regains access.

#### Acceptance Criteria
- Reactivation fails if soft limit is exceeded.
- Action is written to the audit log.

---

### UC-6: Archive Staff

**Actors:** Admin

#### Preconditions
- Staff status is `ACTIVE` or `DISABLED`
- Tenant has not exceeded hard staff limit

#### Main Flow
1. Admin selects staff member.
2. Admin chooses “Archive”.
3. System sets status to `ARCHIVED`.

#### Postconditions
- Staff is permanently removed from active workforce.

#### Acceptance Criteria
- Archived staff cannot access POS.
- Historical data remains intact.
- Action is written to the audit log.

---

### UC-7: Reassign Branch or Role

**Actors:** Admin

#### Preconditions
- Staff status is `ACTIVE` or `DISABLED`
- Target branch exists and is not frozen

#### Main Flow
1. Admin selects staff member.
2. Admin updates:
   - Branch assignment (selected from existing tenant branches)
   - Role
3. System saves changes.

#### Postconditions
- New assignment applies to future operations.

#### Acceptance Criteria
- Historical data remains unchanged.
- Cannot assign to frozen branch.
- If role becomes Manager, visibility is limited to the assigned branch.
- Action is written to the audit log.

---

### UC-8: Assign Staff Shift Schedule

**Actors:** Admin

#### Preconditions
- Admin is authenticated
- Target staff belongs to tenant
- Target branch exists and is **not frozen**
- Staff is assigned to the target branch

#### Main Flow
1. Admin opens staff profile or shift management view.
2. Admin assigns a weekly schedule:
   - Monday–Sunday with start/end times or marked as off.
3. System stores the schedule for that staff + branch.

#### Postconditions
- Staff has an updated weekly shift schedule.

#### Acceptance Criteria
- Only Admin can assign or update shifts.
- Shifts are branch-scoped and stored per staff.
- If a day is marked `off`, no start/end is required.
- Action is written to the audit log.

---

## 7. Data Integrity Rules

- Staff memberships are immutable in identity (user ID never changes).
- Archived staff remain in records for audit/history.
- No hard delete of staff in Capstone I.
- Attendance and sales reference staff by stable IDs.
- Shift schedules are stored per staff and branch; historical attendance remains tied to actual check-in/out timestamps.

---

## 8. Requirements

- R1: Only Admin can invite, disable, archive, reactivate, or reassign staff.
- R2: Staff limits are enforced via:
  - Soft limit = maximum ACTIVE staff memberships
  - Hard limit = maximum ACTIVE + ARCHIVED staff memberships
- R3: Managers have read-only visibility for their branch staff.
- R4: Staff Management does not handle credentials or account identity data.
- R5: All staff lifecycle changes must be audit logged.
- R6: Staff cannot be assigned to a frozen branch.

---

## 9. Acceptance Criteria (Summary)

- AC1: Admin can manage staff lifecycle within quota limits.
- AC2: Staff cannot manage other staff.
- AC3: Archived staff do not break historical records.
- AC4: Disabling staff blocks POS access immediately.
- AC5: Audit log records staff lifecycle changes and assignment changes.
- AC6: Branch assignment is only from system-created branches.

---

## Audit Events Emitted (Minimum)

- `STAFF_INVITED`
- `STAFF_INVITE_REVOKED` (if implemented)
- `STAFF_INVITE_ACCEPTED`
- `STAFF_DISABLED`
- `STAFF_REACTIVATED`
- `STAFF_ARCHIVED`
- `STAFF_ROLE_CHANGED`
- `STAFF_BRANCH_CHANGED`
- `STAFF_SHIFT_ASSIGNED`
