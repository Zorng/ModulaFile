# Tenant Membership Administration Process (Grant / Role Change / Revoke)

## Purpose

This document defines the canonical process for **tenant membership administration**:
- granting membership by phone number (linking or provisioning an identity)
- changing a member’s role
- revoking membership

It exists to keep “who belongs to this business” consistent across IdentityAccess, OrgAccount, and HR.

---

## Why This Process Exists

Real cafés hire, rotate, and remove people frequently.
If membership is not modeled explicitly and enforced consistently:
- access lingers after someone leaves
- authorization becomes inconsistent across modules
- multi-tenant staff turnover creates “duplicate accounts” and password confusion

This process provides one stable orchestration for March MVP.

---

## Participating Domains / Modules

| Artifact | Responsibility |
|---|---|
| Authentication | Resolve or provision `auth_account_id` by phone (no admin-owned passwords) |
| Tenant Membership | Membership lifecycle (ACTIVE/DISABLED/ARCHIVED) + governance kind (`OWNER`/`MEMBER`) + tenant-scoped `role_key` |
| Tenant | Tenant status gates membership changes (future: FROZEN) |
| Access Control | Authorize admin actions + enforce membership changes immediately on sensitive actions |
| Audit (logging) | Traceable membership changes |

---

## Triggers

- Owner/Admin wants to add someone to the tenant (staff or admin)
- Owner/Admin wants to promote/demote someone
- Owner/Admin wants to remove someone from the tenant

---

## Flow A — Grant Membership (Link or Provision Identity)

**Goal:** Add a person to the tenant without creating duplicate identities or setting passwords.

**Inputs:**
- `tenant_id`
- `phone_number`
- `role_key` (March built-ins: `ADMIN`, `MANAGER`, `CASHIER` — extendable later)

**Steps:**
1. **Authorization gate**
   - Actor must be authenticated.
   - Access Control must ALLOW `tenant.membership.grant` (exact action key is implementation detail).
2. **Tenant gate**
   - Tenant exists and is ACTIVE.
3. **Resolve identity**
   - Authentication resolves `auth_account_id` by `phone_number`:
     - if identity exists → reuse it (do not change credentials)
     - if identity does not exist → provision a new AuthenticationAccount (phone only; unverified; no password yet)
4. **Create membership**
   - Create TenantMembership for `(tenant_id, auth_account_id)` with:
     - `status = ACTIVE`
     - `membership_kind = MEMBER` (owners are created during tenant provisioning; ownership transfer is out of scope for March)
     - `role_key = <role_key>`
   - Enforce uniqueness: no duplicate memberships for the same identity within a tenant.
5. **Audit**
   - Record MEMBER_GRANTED (who granted, which role, when).

**Rules:**
- Admins/owners never set or know passwords.
- Provisioning an identity must not auto-reset an existing identity’s password.
- If membership already exists and is ACTIVE:
  - idempotent no-op if role is unchanged
  - otherwise treat as Flow B (role change), not as a duplicate-create.

**Notes (important):**
- StaffProfile creation and branch assignment are owned by HR workflows (see staff provisioning orchestration).
- This process only establishes tenant-scoped access facts.

---

## Flow B — Change Member Role

**Goal:** Promote/demote a member’s authorization role within the tenant.

**Inputs:**
- `tenant_id`
- `auth_account_id` (or member reference)
- `new_role_key`

**Steps:**
1. Authorization gate: Access Control must ALLOW `tenant.membership.changeRole`.
2. Validate member exists and is not ARCHIVED.
3. Validate `new_role_key` is recognized by the current RolePolicy (do not hardcode role lists in processes).
4. Enforce governance invariant:
   - If `membership_kind = OWNER`, the role key must not be less powerful than `ADMIN` (for March: keep `role_key = ADMIN` for owners).
5. Update `role_key`.
6. Audit MEMBER_ROLE_CHANGED.

**Rule: Immediate effect**
- Role changes must affect the next authorization decision (server-side source of truth).

---

## Flow C — Revoke Membership (Archive)

**Goal:** Remove access while preserving history.

**Inputs:**
- `tenant_id`
- `auth_account_id`

**Steps:**
1. Authorization gate: Access Control must ALLOW `tenant.membership.revoke`.
2. Validate member exists and is not already ARCHIVED.
3. Enforce safety invariant: cannot archive the last ACTIVE `OWNER` membership for the tenant.
4. Set membership `status = ARCHIVED`.
5. Audit MEMBER_REVOKED.

**Rule: Immediate effect**
- Access Control must deny the next sensitive request in that tenant context.

**March-safe note**
- A revoked membership does not delete operational history (attendance, sales, audits remain traceable).

---

## Enforcement Expectations (Cross-Layer)

- Access Control must re-check membership status/`role_key` on sensitive actions (START_WORK, END_WORK, finalize sale, void approve, open/close cash session).
- If a membership becomes ARCHIVED mid-session:
  - the next sensitive action must fail closed
  - the user must be able to switch to another ACTIVE tenant membership (if any)

---

## Failure Modes (Behavioral)

- TENANT_NOT_FOUND
- TENANT_NOT_ACTIVE
- PHONE_INVALID
- MEMBER_NOT_FOUND
- MEMBER_ARCHIVED
- ROLE_KEY_INVALID
- DUPLICATE_MEMBERSHIP
- CANNOT_REMOVE_LAST_OWNER
- CANNOT_DEMOTE_OWNER_ROLE

---

## Related Docs

- Domain: `BusinessLogic/2_domain/10_Identity&Authorization/tenant_membership_domain.md`
- Domain: `BusinessLogic/2_domain/10_Identity&Authorization/authentication_domain_consistency_patched.md`
- ModSpec: `BusinessLogic/5_modSpec/20_OrgAccount/tenant_module.md`
- Process: `BusinessLogic/4_process/20_IdentityAccess/10_identity_activation_recovery_orchestration.md`
- Contract: `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md`

_End of Tenant Membership Administration Process_
