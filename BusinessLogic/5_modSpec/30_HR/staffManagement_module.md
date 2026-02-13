# staffManagement_module.md

## Module: Staff Management

**Version:** 3.1  
**Status:** Patched (simplified onboarding; user-owned credentials; aligned with Authentication, Access Control, Tenant, and Operator Seats)  
**Module Type:** Supporting Domain (Org Operations / HR-light)  
**Depends on:** Authentication (identity & credentials), Tenant (membership facts: `membership_kind` + `role_key`), Branch (branch entity), Access Control (authorization), Attendance (signals), Audit (logging)  
**Related Modules:** Attendance, Cash Session, Sale, Access Control, Subscription/Entitlements

---

## 1. Purpose

Staff Management manages the business-facing representation of people who operate Modula within a tenant:
- staff profiles and lifecycle,
- explicit branch assignments (authorization facts),
- operational readiness of staff,
- alignment with **Operator Seat** limits (commercial concurrency guard rail).

To prioritize shipping and reduce onboarding complexity, Modula adopts **explicit invite + accept** onboarding with user-owned credentials.

---

## 2. Key Boundary Clarifications

### 2.1 Authentication vs Staff Management

- **Authentication** owns credentials (phone number + password), sessions, and password security.
- **Staff Management** owns staff profiles, lifecycle, and branch assignments.

**Critical rule**
- Admins/owners must **never** set, view, or reset staff passwords.
- Passwords are owned by Authentication and are always set via Authentication-managed self-service workflows (OTP verification + password setup/reset).
- Staff Management may trigger Authentication to send an OTP for credential setup, but it does not handle credentials directly.

---

### 2.2 Identity Mapping Contract

- Each StaffProfile is linked to exactly one `auth_account_id`.
- Authentication uses **phone number + password** as the credential.
- All authorization and licensing checks use `auth_account_id` as the actor key.

---

### 2.3 Tenant Membership vs Staff Profile

- **Tenant module** owns tenant membership facts (`membership_kind`, `role_key`, membership status).
- **Staff Management** owns staff profiles and branch assignments.

A staff member can operate only if all are true:
- authenticated (Authentication),
- ACTIVE staff profile (this module),
- ACTIVE tenant membership (Tenant),
- ACTIVE branch assignment (this module),
- authorization decision is ALLOW (Access Control),
- operator seats are available (Subscription & Entitlements).

**Entitlements note (billing guard rail)**
- When Workforce is not subscribed for a branch, Staff Management becomes read-only for that branch:
  - viewing history may remain available,
  - but staff lifecycle and branch assignment writes are blocked.
- Entitlement key (branch-scoped): `module.workforce`

---

## 2.4 Fair-Use Limits (Safety, Not Pricing)

Staff creation and management can be abused by automation (e.g., creating extreme numbers of staff profiles).
Modula enforces technical safety limits as guard rails:
- these limits are not monetization and must not be framed as "upgrade plan",
- they exist to prevent resource exhaustion (accidental or malicious).

Canonical references:
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/fair_use_limits_domain.md`
- Gate process: `BusinessLogic/4_process/60_PlatformSystems/85_fair_use_limit_gate_process.md`

---

## 3. Staff Lifecycle (Simplified)

INVITED is a **membership** state, not a staff profile state.  
Staff profiles are created only after an invitation is accepted.

**Lifecycle**
- `ACTIVE` → `DISABLED` → `ARCHIVED`

### Semantics
- ACTIVE: staff may operate (subject to authorization and licensing)
- DISABLED: immediate operational block; history preserved
- ARCHIVED: removed from active UI lists; immutable historical reference

---

## 4. Core Concepts & Data Model

### 4.1 StaffProfile
Represents a staff member within a tenant.

Minimum fields:
- `tenant_id`
- `auth_account_id`
- `display_name`
- `staff_code` (optional)
- `job_title` (optional; display/scheduling context only)
- `pin_hash` (optional; for quick unlock)
- `status` = ACTIVE | DISABLED | ARCHIVED
- `created_at`, `updated_at`

**Notes**
- StaffProfile does not store login credentials.
- Disabling staff must take effect immediately for authorization.

---

### 4.2 BranchAssignment (Authorization Fact)
A first-class, explicit fact consumed by Access Control.

Minimum fields:
- `tenant_id`
- `branch_id`
- `auth_account_id`
- `assignment_status` = ACTIVE | REVOKED
- `assigned_at`
- `revoked_at?`
- `assigned_by` (auth_account_id)
- `reason?` (optional)

**Design rule**
- Branch access is granted **only** via explicit assignments (including for ADMIN/MANAGER).
- Revocation takes effect on the next request.

---

### 4.3 Concurrent Operator (Logical Concept)

A staff member consumes an operator seat when they **start work** at a branch:
- `START_WORK` creates an ACTIVE attendance session,
- the active attendance session is the canonical "active operator" signal for seat consumption.

The storage mechanism is an implementation detail; the definition is canonical.

---

## 5. Invariants

- INV-SM1: Each StaffProfile maps 1:1 to an `auth_account_id`.
- INV-SM2: DISABLED or ARCHIVED staff cannot perform operational actions.
- INV-SM3: Branch access requires explicit BranchAssignment.
- INV-SM4: BranchAssignment revocation is immediate.
- INV-SM5: Operator seat limits gate operation, not staff record creation.
- INV-SM6: `role_key` does not bypass branch assignment or licensing limits (including for ADMIN/MANAGER).
- INV-SM7: Staff lifecycle and branch assignment writes must pass the platform idempotency gate:
  - `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`

---

## 6. Self-Contained Processes

### UC-SM1 — Invite Staff Member (Owner/Admin)
**Goal:** Owner/admin invites a person to join the tenant without sharing credentials.

Canonical process reference:
- `BusinessLogic/4_process/10_WorkForce/05_staff_provisioning_orchestration.md`

**Inputs**
- tenant_id
- phone_number (login identifier)
- initial `role_key` (Tenant membership)
- intended branch assignments (one or more)
- optional display_name

**Steps**
1. Resolve Authentication account by phone number:
   - If it exists, reuse it (do not change credentials).
   - Otherwise provision an Authentication account (phone identifier only; unverified; no password yet).
2. Create or update TenantMembership with:
   - `membership_status = INVITED`
   - `membership_kind = MEMBER` (staff provisioning does not create owners)
   - `role_key = <role_key>`
3. Record intended branch assignments (pending until acceptance).
4. Optionally send invitation notification (SMS/in-app).

**Rules**
- Password is never stored or retrievable by Staff Management.
- Invite does not fail just because phone number already exists globally.
- Invite fails if the staff member already has an ACTIVE membership in this tenant (no duplicates).
- Invite does not consume a concurrent staff slot.
- StaffProfile and BranchAssignment are created only after acceptance.

**Audit**
-- STAFF_INVITED

---

### UC-SM2 — Accept Invitation (Self-Service)
**Goal:** Invited user activates access and becomes operational.

**Steps**
1. Authentication handles profile fields + password set + OTP verification (if needed).
2. Set TenantMembership to `ACTIVE` and record acceptance.
3. Create StaffProfile linked to `auth_account_id` with status ACTIVE.
4. Create BranchAssignment(s) from the pending branch list.

**Audit**
- STAFF_INVITE_ACCEPTED
- STAFF_PROFILE_CREATED

---

### UC-SM3 — Disable Staff (Immediate Block)
**Goal:** Immediately prevent staff from operating.

**Steps**
1. Set StaffProfile status → DISABLED.
2. Access Control must deny next operational request.
3. Active operational sessions should be terminated if possible.

**Audit**
- STAFF_DISABLED

---

### UC-SM4 — Archive Staff
**Goal:** Remove staff from active operations while preserving history.

**Rules**
- ARCHIVED staff cannot be reactivated.
- Historical references remain valid.

**Audit**
- STAFF_ARCHIVED

---

## 7. Branch Assignment Processes

### UC-SM4 — Assign Staff to Branch
**Goal:** Grant operational access to a branch.

**Rules**
- Branch must be ACTIVE.
- Assignment is explicit and idempotent.

**Audit**
- BRANCH_ACCESS_GRANTED

---

### UC-SM5 — Revoke Staff from Branch
**Goal:** Remove operational access to a branch.

**Rules**
- Revocation takes effect immediately.
- If staff is operating in that branch, next request is denied.

**Audit**
- BRANCH_ACCESS_REVOKED

---

## 8. Operator Seat Enforcement

### 8.1 Enforcement Points
Operator seats are enforced at:
- `START_WORK` (Work Start orchestration)

If limit reached:
- deny with `SEAT_LIMIT_REACHED`
- no silent fallback

### 8.2 What Does Not Count
- inactive sessions
- DISABLED / ARCHIVED staff
- staff profiles without an ACTIVE attendance session

---

## 9. Read APIs (Facts for Access Control)

- Get StaffProfile status by (tenant_id, auth_account_id)
- IsAssignedToBranch(tenant_id, branch_id, auth_account_id)
- ListBranchesForStaff(tenant_id, auth_account_id)

**Index recommendations**
- Unique: (tenant_id, branch_id, auth_account_id)
- Lookup: (tenant_id, auth_account_id)

---

## 10. Failure Modes

- STAFF_ALREADY_EXISTS
- STAFF_NOT_ACTIVE
- NO_BRANCH_ASSIGNMENT
- BRANCH_NOT_ACTIVE
- SEAT_LIMIT_REACHED

---

## 11. Audit & Observability

Log:
- STAFF_ACCOUNT_CREATED
- STAFF_DISABLED
- STAFF_ARCHIVED
- BRANCH_ACCESS_GRANTED / REVOKED
- STAFF_LIMIT_REACHED

---

## 12. Out of Scope (March MVP)

- Invite-based onboarding
- Email-based login
- Payroll / HR documents
- Billing payment collection automation (entitlement enforcement is in-scope via Access Control)
- Multi-tenant franchise hierarchies

---

## 13. Integration Notes

- Authentication handles password security and reset flows.
- Access Control enforces permissions using tenant membership + branch assignments.
- Operator seat gating enforces concurrency limits using Attendance sessions as the canonical active-operator signal.
