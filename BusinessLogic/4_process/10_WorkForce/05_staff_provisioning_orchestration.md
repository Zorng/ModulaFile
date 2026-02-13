# Staff Invitation + Onboarding Orchestration (Owner/Admin)

## Purpose

This document defines the cross-domain orchestration for **owner/admin inviting staff** and the **self-service acceptance** that follows:
- link or provision the global identity (phone-based)
- create tenant membership in `INVITED`
- accept invitation (OTP + profile + password)
- create the staff profile
- assign the staff member to one or more branches

This process exists so onboarding is fast for cafes, while credentials remain **user-owned**.

---

## Why This Process Exists

In real stores, the hiring decision is already made before the system is used.
Owners expect to set up staff quickly without password sharing.

At the same time:
- a person may move between tenants over time (multi-tenant SaaS)
- admins must not reset or control staff passwords
- access facts must be explicit so Access Control can enforce them deterministically

So onboarding must be a single, safe orchestration — not scattered rules across modules.

---

## Participating Domains / Modules

| Artifact | Responsibility |
|---|---|
| Authentication | Resolve/provision identity by phone; OTP activation; profile + password set |
| Tenant Membership | Tenant-scoped membership facts (`membership_kind`, `role_key`, status) |
| Staff Profile & Assignment | Staff profile lifecycle + branch assignments |
| Branch | Branch status gates assignment and later operation |
| Access Control | Authorize provisioning actions; enforce branch access |
| Audit (logging) | Traceable provisioning events |

---

## Triggers

This orchestration begins when:
- an Owner/Admin invites a staff member, or
- an invited staff member accepts the invitation

---

## Flow A — Invite Staff (Owner/Admin)

**Goal:** Create a tenant invitation and prepare for onboarding without sharing credentials.

**Inputs:**
- phone number
- `role_key` (March built-ins: `ADMIN`, `MANAGER`, `CASHIER` — extendable later)
- one or more branch assignments (intended)
- optional display name (if known)

### Step 1 — Authorization Gate
Access Control must ALLOW the actor to invite staff for this tenant.

### Step 2 — Tenant and Branch Validation
Validate:
- Tenant exists and is ACTIVE.
- Each target branch exists and is `ACTIVE` (not FROZEN).

If any branch is not ACTIVE, fail with a clear reason.

### Step 3 — Resolve or Provision Global Identity (Phone)
Authentication resolves `auth_account_id` by phone:
- If identity exists → reuse it (do not change credentials).
- If identity does not exist → provision an AuthenticationAccount (phone only; unverified; no password yet; profile fields not set).

**Rule:** provisioning must never create a “first-time password” known by the admin.

### Step 4 — Create INVITED Tenant Membership
Create or update TenantMembership for `(tenant_id, auth_account_id)`:
- `membership_status = INVITED`
- `membership_kind = MEMBER` (ownership/transfer is out of scope for March)
- `role_key = <role_key>`
- record `invited_by` and `invited_at`

If membership already exists:
- if ACTIVE → treat as role-change, not invitation
- if INVITED → idempotent update of role_key/metadata
- if ARCHIVED → require re-invite (new invitation intent)

Reference process:
- `BusinessLogic/4_process/20_OrgAccount/10_tenant_membership_administration_process.md`

### Step 5 — Record Intended Branch Assignments (Pending)
Store the intended branch list for use after acceptance.

Implementation detail: this can be stored as invitation metadata or a pending assignment list.

### Step 6 — Send Invitation (Best-Effort)
Optionally send an invite notification (SMS/in-app).
Onboarding must not fail solely due to SMS delivery issues.

---

## Flow B — Accept Invitation (Self-Service)

**Goal:** The invited person activates access and becomes operational.

### Step 1 — Authentication Activation
Authentication handles:
- basic profile fields (first name, last name, gender, DOB)
- password setup (validate policy)
- OTP verification (phone ownership)

If the identity was already verified, skip OTP and only ensure profile completeness if required.

### Step 2 — Accept Membership
Update TenantMembership to:
- `membership_status = ACTIVE`
- record `accepted_at`

### Step 3 — Create Staff Profile
Create StaffProfile for `(tenant_id, auth_account_id)`:
- `display_name` (prefer AuthAccount profile name; fallback to invite display name)
- `status = ACTIVE`

Do not create duplicate staff profiles for the same identity in the same tenant.

### Step 4 — Create Branch Assignments
Create BranchAssignment(s) using the pending branch list:
- `status = ACTIVE`
- record who assigned and when (auditability)

If multiple branches are assigned, branch context must be selected before work begins.

### Step 5 — Audit Logging
Record:
- invite accepted (who, tenant, role_key)
- staff profile created
- branch access granted

---

## Failure Handling

On failure:
- do not create partial staff facts unless explicitly defined
- return a clear reason

Typical failures:
- TENANT_NOT_ACTIVE
- BRANCH_NOT_ACTIVE
- PHONE_INVALID
- INVITE_NOT_FOUND / INVITE_EXPIRED
- ROLE_KEY_INVALID

---

## Key Guarantees

- Onboarding is explicit (invite + accept).
- Credentials are always user-owned (OTP activation/reset; no admin-known passwords).
- Multi-tenant identities are reused safely across businesses.
- Branch access is explicit and enforceable (no implicit “admins can access all branches”).
- Staff profiles and branch assignments are created only after acceptance.

---

## Related Docs

- Story: `BusinessLogic/1_stories/handling_staff&attendance/setup_staff.md`
- Domain: `BusinessLogic/2_domain/10_Identity&Authorization/tenant_membership_domain.md`
- Domain: `BusinessLogic/2_domain/30_HR/staff_profile_and_assignment_domain.md`
- Process: `BusinessLogic/4_process/20_OrgAccount/10_tenant_membership_administration_process.md`
- Process: `BusinessLogic/4_process/20_IdentityAccess/10_identity_activation_recovery_orchestration.md`
- ModSpec: `BusinessLogic/5_modSpec/30_HR/staffManagement_module.md`

_End of Staff Invitation + Onboarding Orchestration_
