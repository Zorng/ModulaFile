# Identity Activation + Recovery Orchestration (OTP + Context Selection)

## Purpose

This document defines the canonical behavior for:
- activating a **provisioned** user (phone exists, no password yet)
- password recovery (OTP reset)
- resolving **tenant + branch context** after authentication

This is a **behavior contract** (process layer). It does not prescribe UI layouts or screens.

---

## Why This Process Exists

Modula is SaaS-first:
- one identity may belong to multiple tenants
- tenant owners/admins provision access (membership), but **do not own credentials**
- staff turnover and multi-tenant work must not create “new accounts” per tenant

So we need one stable definition of how a person:
1) proves phone ownership (OTP),
2) sets/resets their password (self-service),
3) selects where they are working (tenant/branch context),
without any tenant ever “owning” the identity’s password.

---

## Participating Domains / Modules

| Artifact | Responsibility |
|---|---|
| Authentication | OTP verification, password set/reset, session issuance/revocation |
| Tenant Membership | Which tenants the identity can operate in (ACTIVE memberships only) |
| Staff Profile & Assignment | Which branches the identity can operate in (branch assignments) |
| Branch | Branch status (ACTIVE/FROZEN) gates branch-scoped work |
| Access Control | Authorization re-check on sensitive actions (fail closed) |
| Audit (logging) | Traceable auth + context selection events |

---

## Triggers

This orchestration begins when:
- a user chooses **Activate account** (provisioned user) or **Forgot password**
- a user successfully authenticates and must establish working context (tenant/branch)

---

## Flow A — Activate Provisioned Account (OTP → Set Password)

**Goal:** A provisioned user sets their first password using OTP verification.

**Preconditions:**
- User has a phone number.
- An AuthenticationAccount may already exist (provisioned by onboarding), but may not have a password yet.

**Steps:**
1. User submits phone number.
2. Authentication resolves the identity by phone:
   - if it exists → continue
   - if it does not exist → allow self-registration (but tenant access may be empty)
3. Authentication sends OTP to the phone (rate-limited).
4. User submits OTP.
5. Authentication verifies OTP and marks the phone as verified.
6. Authentication accepts a **new password** and stores it (password is now set).
7. Authentication issues a session for the identity.
8. Continue to **Flow C** (tenant context resolution).

**Rules:**
- Admins/owners never set or know passwords (credential ownership stays with the person).
- If the identity already has a password, “activation” is not needed; direct the user to login or reset.
- OTP send/verify may be retried under security limits; repeated activation attempts must not create duplicate identities.

---

## Flow B — Forgot Password (OTP → Reset Password)

**Goal:** A user regains access without any admin involvement.

**Steps:**
1. User submits phone number.
2. Authentication sends OTP (rate-limited).
3. User submits OTP.
4. Authentication verifies OTP.
5. User sets a new password.
6. Authentication revokes all active sessions for that identity (security boundary).
7. Authentication issues a new session (or returns to login — implementation choice).
8. Continue to **Flow C** (tenant context resolution).

---

## Flow C — Resolve Tenant Context (Post-Auth)

**Goal:** Establish which tenant the session is operating in.

**Inputs:**
- `auth_account_id` (from Authentication session)

**Steps:**
1. List tenant memberships for `auth_account_id`.
2. Filter to **ACTIVE** memberships only.
3. Resolve:
   - **0 active memberships** → session remains global-only; user cannot operate tenant-scoped features.
   - **1 active membership** → auto-select tenant context.
   - **2+ active memberships** → user must choose; validate membership is still ACTIVE at selection time.
4. Store `tenant_id` as session context (or return it as an explicit “working context” output).
5. Continue to **Flow D** (branch context resolution).

**Rules:**
- Tenant selection does not grant access; it only selects among ACTIVE memberships.
- Membership may change at any time; Access Control must re-check on sensitive actions (see Related Contracts).

---

## Flow D — Resolve Branch Context (Within Tenant)

**Goal:** Establish which branch the session is operating in (when branch-scoped work exists).

**Inputs:**
- `auth_account_id`
- `tenant_id`

**Steps:**
1. List branch assignments for this identity within the selected tenant.
2. Filter to ACTIVE assignments and ACTIVE branches.
3. Resolve:
   - **0 eligible branches** → user cannot operate branch-scoped features; show a clear “no branch assigned” state.
   - **1 eligible branch** → auto-select branch context.
   - **2+ eligible branches** → user must choose.
4. Store `branch_id` as session context (or return it as an explicit output).

**Rules:**
- Branch context is required for operational actions (sale, cash session, attendance work start/end).
- If branch is FROZEN, it must not be selectable for operational work.

---

## Failure Modes (Behavioral)

Typical outcomes the client must support:
- phone not found (only for login; activation/reset can still proceed as “register/claim”)
- auth account disabled
- OTP rate-limited / expired / invalid
- password policy violation (too weak)
- no active tenant memberships
- tenant not active (future: FROZEN)
- no branch assignment for selected tenant
- branch not active (FROZEN)

---

## Key Guarantees

- Credential ownership is always user-owned (OTP + password lifecycle is Authentication-owned).
- Provisioning access never requires an admin-known “first password”.
- Tenant/branch context selection is deterministic and validated against current facts.
- Access can be removed mid-session; sensitive actions must re-check authorization and fail closed.

---

## Related Docs

- Domain: `BusinessLogic/2_domain/10_Identity&Authorization/authentication_domain_consistency_patched.md`
- Domain: `BusinessLogic/2_domain/10_Identity&Authorization/tenant_membership_domain.md`
- Domain: `BusinessLogic/2_domain/30_HR/staff_profile_and_assignment_domain.md`
- Contract: `BusinessLogic/3_contract/20_ux_specs/identity_tenant_workflow_ux_spec.md`
- Contract: `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md`
- Process: `BusinessLogic/4_process/10_WorkForce/10_work_start_end_orchestration.md`

_End of Identity Activation + Recovery Orchestration_
