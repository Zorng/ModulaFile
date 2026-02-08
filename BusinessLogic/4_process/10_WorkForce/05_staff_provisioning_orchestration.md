# Staff Provisioning Orchestration (Owner-Provisioned Onboarding)

## Purpose

This document defines the cross-domain orchestration for **owner/admin provisioning** of staff:
- link or create the global identity (phone-based)
- grant tenant membership + `role_key`
- create the staff profile
- assign the staff member to one or more branches

This process exists so onboarding is fast for cafés, while credentials remain **user-owned**.

---

## Why This Process Exists

In real stores, the hiring decision is already made before the system is used.
Owners expect to set up staff quickly without invitations, approvals, or password sharing.

At the same time:
- a person may move between tenants over time (multi-tenant SaaS)
- admins must not reset or control staff passwords
- access facts must be explicit so Access Control can enforce them deterministically

So onboarding must be a single, safe orchestration — not scattered rules across modules.

---

## Participating Domains / Modules

| Artifact | Responsibility |
|---|---|
| Authentication | Resolve/provision identity by phone; OTP activation is self-service |
| Tenant Membership | Tenant-scoped membership facts (`membership_kind`, `role_key`, status) |
| Staff Profile & Assignment | Staff profile lifecycle + branch assignments |
| Branch | Branch status gates assignment and later operation |
| Access Control | Authorize provisioning actions; enforce branch access |
| Audit (logging) | Traceable provisioning events |

---

## Trigger

The process begins when an Owner/Admin attempts to:

> “Add a staff member to my business”

Inputs typically include:
- phone number
- display name
- `role_key` (March built-ins: `ADMIN`, `MANAGER`, `CASHIER` — extendable later)
- one or more branch assignments

---

## Step-by-Step Flow

## Step 1 — Authorization Gate

Access Control must ALLOW the actor to provision staff for this tenant.

---

## Step 2 — Tenant and Branch Validation

Validate:
- Tenant exists and is ACTIVE.
- Each target branch exists and is `ACTIVE` (not FROZEN).

If any branch is not ACTIVE, fail with a clear reason (no partial assignment for March unless explicitly supported).

---

## Step 3 — Resolve or Provision Global Identity (Phone)

Authentication resolves `auth_account_id` by phone:
- If identity exists → reuse it (do not change credentials).
- If identity does not exist → provision an AuthenticationAccount (phone only; unverified; no password yet).

**Rule:** provisioning must never create a “first-time password” known by the admin.

---

## Step 4 — Grant Tenant Membership + Role Key

Create or update TenantMembership facts for `(tenant_id, auth_account_id)`:
- membership `status = ACTIVE`
- membership `membership_kind = MEMBER` (ownership/transfer is out of scope for March)
- membership `role_key = <role_key>`

If membership already exists:
- treat as idempotent if role is unchanged
- otherwise treat as an explicit role-change operation (do not create duplicates)

Reference process:
- `BusinessLogic/4_process/20_OrgAccount/10_tenant_membership_administration_process.md`

---

## Step 5 — Create Staff Profile (Tenant-scoped)

Create StaffProfile for `(tenant_id, auth_account_id)` with:
- `display_name`
- `status = ACTIVE`

**Rule:** do not create duplicate staff profiles for the same identity in the same tenant.

---

## Step 6 — Create Branch Assignments

Create BranchAssignment(s) for each target branch:
- `status = ACTIVE`
- record who assigned and when (auditability)

Multi-branch assignment is supported.
If multiple branches are assigned, branch context must be selected before work begins.

---

## Step 7 — Activation Is Self-Service (Optional Best-Effort Signal)

If the identity is newly provisioned or has no password yet:
- Authentication supports OTP-based activation/reset.
- The system may optionally send an OTP prompt/SMS, but onboarding must not fail solely due to SMS delivery issues.

Reference process:
- `BusinessLogic/4_process/20_IdentityAccess/10_identity_activation_recovery_orchestration.md`

---

## Step 8 — Audit Logging

Record:
- staff provisioned (who did it, which tenant, which `role_key`)
- branch access granted
- membership granted (`membership_kind = MEMBER`) and `role_key` set

---

## Failure Handling

On failure:
- do not create partial staff facts unless explicitly defined
- return a clear reason

Typical failures:
- TENANT_NOT_ACTIVE
- BRANCH_NOT_ACTIVE
- PHONE_INVALID
- STAFF_ALREADY_EXISTS (profile already exists in this tenant)
- ROLE_KEY_INVALID

---

## Key Guarantees

- Owner-provisioned onboarding stays fast (no invite/acceptance workflow for March).
- Credentials are always user-owned (OTP activation/reset; no admin-known passwords).
- Multi-tenant identities are reused safely across businesses.
- Branch access is explicit and enforceable (no implicit “admins can access all branches”).

---

## Related Docs

- Story: `BusinessLogic/1_stories/handling_staff&attendance/setup_staff.md`
- Domain: `BusinessLogic/2_domain/10_Identity&Authorization/tenant_membership_domain.md`
- Domain: `BusinessLogic/2_domain/30_HR/staff_profile_and_assignment_domain.md`
- ModSpec: `BusinessLogic/5_modSpec/30_HR/staffManagement_module.md`

_End of Staff Provisioning Orchestration_
