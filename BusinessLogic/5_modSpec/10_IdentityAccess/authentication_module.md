# authentication_module.md

## Module: Authentication (Core Module)

**Version:** 2.0  
**Status:** Patched (Renamed from `auth_module`; explicit AuthN vs AuthZ separation)  
**Module Type:** Core Module  
**Depends on:** Tenant & Branch (context), Staff Membership / Roles (IdentityAccess), Audit Logging (Platform), Policy & Configuration (Platform), Offline Sync (Platform)  
**Related Modules:** Tenant, Branch, Staff Management, Attendance, Sale, Cash Session, Receipt, Audit, Policy, Offline Sync

---

## 1. Purpose

The **Authentication** module exists to answer one question reliably:

- **Who is this user, and are they authenticated right now?**

It manages:
- identity login methods (Phase 0: phone-first)
- credential lifecycle (password set/reset)
- session lifecycle (issue/refresh/revoke)
- security controls (OTP verification, lockouts, rate limits)

**Fair-use note (safety, not pricing)**
- Authentication rate limits and lockouts are security/fair-use controls.
- They must never be treated as subscription monetization.
- Canonical reference: `BusinessLogic/2_domain/60_PlatformSystems/fair_use_limits_domain.md`

### What this module intentionally does *not* own

This module does **not** own:
- tenant creation / provisioning
- staff onboarding or termination
- role policy design
- subscription/billing/capabilities

Those belong to their respective domains/modules.

---

## 2. Important Clarification: Authentication vs Authorization

Modula needs both:

### Authentication (AuthN) — *owned here*
- Login, register, verify identity
- Issue sessions/tokens
- Manage password/OTP recovery
- Revoke sessions on logout/reset/disable

### Authorization (AuthZ) — *used by Modula, but not owned here*
- Decide whether an authenticated user can perform an action
- Based on:
  - Tenant membership
  - Roles/permissions
  - Branch assignment rules (if any)
  - (Future) Tenant status and entitlements/capabilities

#### Why we separate them explicitly (even if some checks live here today)

Because mixing them causes long-term confusion and coupling:
- teams start treating “Auth” as “everything about access”
- billing/capability rules leak into login flows
- staff lifecycle and tenant provisioning accidentally end up in the wrong module

So in this spec:
- **AuthN is the module’s responsibility**
- **AuthZ is treated as a gateway decision that consumes external facts**
  (membership/roles, tenant status, entitlements)

This keeps Modula SaaS-ready without slowing the March MVP.

---

## 3. Scope & Responsibilities

### 3.1 Owned (AuthN responsibilities)
- phone-first registration and verification
- login with credentials
- session/token issuance and refresh
- logout and session revocation
- password change (logged-in)
- password recovery via OTP
- account disable/enable (if needed in Capstone scope)
- audit events for auth/security actions

### 3.2 Authorization Gateway (AuthZ decision surface)
This module may expose a convenience endpoint/function to:
- validate tenant context selection
- validate branch context selection
- answer “is this action allowed?” by consulting external sources

But:
- it **does not own** the truth of roles/memberships
- it **does not define** role matrices or approval policy
- it must remain compatible with a future **Entitlements** snapshot

---

## 4. Ubiquitous Language

- **AuthAccount**: a global identity record (human) that can authenticate
- **Identifier**: phone number (Phase 0) or email (future)
- **Credential**: password hash + OTP verification state
- **Session**: authenticated session (access/refresh tokens, revocation state)
- **Tenant Context**: which tenant/workspace the user is operating in
- **Branch Context**: which branch/location the user is operating in (if required)
- **Membership**: link between AuthAccount and Tenant (owned by Tenant Membership)
- **Membership Kind**: governance relationship (OWNER/MEMBER) (owned by Tenant Membership)
- **Role Key**: tenant-scoped authorization role key (`role_key`; e.g., `ADMIN`, `MANAGER`, `CASHIER`, ...) (owned by Tenant Membership)
- **Authorization Decision**: yes/no answer using external facts

---

## 5. Use Cases

> Canonical process references (behavior source of truth):
> - Identity activation + recovery + context selection: `BusinessLogic/4_process/20_IdentityAccess/10_identity_activation_recovery_orchestration.md`
> - Staff onboarding that provisions identities: `BusinessLogic/4_process/10_WorkForce/05_staff_provisioning_orchestration.md`

### 5.1 Authentication (Owned)

#### UC-1: Register Account (Phone-first with SMS OTP)
**Goal:** Create an AuthAccount and verify the phone number.  
**Primary actor:** user  
**Preconditions:**
- phone number not already registered (or follow “recover/merge” rule)
**Main flow:**
1. User submits phone number
2. System sends OTP
3. User submits OTP
4. System creates AuthAccount (ACTIVE) and marks identifier as verified
5. User sets initial password (or password is set in a separate step)
**Postconditions:**
- AuthAccount exists
- phone identifier verified
- ready for login

---

#### UC-2: Login
**Goal:** Authenticate user and create a session.  
**Main flow:**
1. User submits phone + password
2. System validates credentials
3. System issues session (access + refresh tokens)
4. Audit login success/failure

---

#### UC-6: Logout
**Goal:** End current device session.  
**Main flow:**
1. User requests logout
2. System revokes current session/refresh token
3. Audit logout

---

#### UC-7: Change Password (Logged-in)
**Goal:** Replace password while authenticated.  
**Main flow:**
1. User provides current password + new password
2. System validates current password
3. System updates password hash
4. System revokes other sessions (optional policy)
5. Audit password change

---

#### UC-8: Forgot Password (SMS OTP Recovery)
**Goal:** Recover access when user forgot password.  
**Main flow:**
1. User submits phone number
2. System sends OTP
3. User submits OTP
4. User sets new password
5. System revokes existing sessions (security boundary)
6. Audit password reset

---

#### UC-9: Refresh Session (Token Refresh)
**Goal:** Keep the session alive without re-entering credentials.  
**Main flow:**
1. Client submits refresh token
2. System validates refresh token
3. System issues a new access token (and rotates refresh token if enabled)
4. Audit refresh

**Notes (important):**
- refresh must be safe under concurrency
- prefer refresh-token rotation to prevent token replay

---

### 5.2 Authorization Gateway (Consumes External Facts)

> These use cases exist because Modula is multi-tenant and often branch-scoped.  
> However, this module **does not own** membership/role truth; it only checks it.

#### UC-3: Select Tenant (Multi-tenant per account)
**Goal:** Choose which tenant the user is operating in.  
**Main flow:**
1. System lists tenants available to the AuthAccount (from Membership source)
2. User selects a tenant
3. System stores tenant context in the session (or returns it to the client)

**Owned vs consumed:**
- Auth stores “selected tenant context” in session
- Membership truth is consumed from Staff/Tenant

---

#### UC-4: Resolve Branch Context
**Goal:** Choose/validate which branch the user is operating in.  
**Main flow:**
1. System lists available branches for this user in this tenant (based on assignment rule)
2. User selects branch
3. System stores branch context in the session (or returns it)

**Notes:**
- Branch eligibility is determined by **explicit branch assignments** (no implicit "all branches" access).
- Auth validates using assignment truth owned by Staff Management / Org domains.

---

#### UC-5: Authorize Request (Role-based Authorization)
**Goal:** Decide whether an authenticated request is allowed.  
**Inputs:**
- actor (auth_account_id)
- tenant_context (tenant_id)
- branch_context (optional)
- action (string/enum)
**Decision sources (consumed):**
- Membership + role assignment (Staff/Tenant)
- Policy configuration (Policy module)
- (Future) Tenant status (Tenant module) and entitlements snapshot (Subscription/Entitlements)

**Output:**
- ALLOW / DENY (+ reason code)

**Important:** This is a *decision boundary*, not a place to encode business logic.

---

## 6. Data & State

### Stored by Authentication
- AuthAccount
- Credential state (password hash, OTP verification timestamps)
- Session records (refresh token hash, revocation timestamp)

### Stored by other modules (referenced)
- Tenant membership, roles, branch assignments
- Tenant status (ACTIVE/FROZEN)
- Entitlements snapshot (future)

---

## 7. Audit & Observability

Authentication must emit audit-grade events for:
- login success/failure
- OTP verification success/failure
- password change/reset
- logout
- session refresh/revoke
- account disabled/enabled (if supported)

This supports incident response and sensitive workflow accountability.

---

## 8. Offline Considerations

- Auth normally requires connectivity for OTP verification.
- Offline-first POS should avoid requiring online auth checks per transaction.
- Use long-lived refresh sessions with reasonable expiry and re-auth rules.
- Authorization should remain enforceable using cached membership/entitlements where applicable (future).

---

## 9. Failure Modes

- OTP provider down → registration/recovery blocked; clear UX, retry policy
- brute force attempts → rate limit + lockout policy
- refresh token replay → rotate refresh token and revoke suspected sessions
- disabled account → deny login and revoke sessions

---

## 10. Out of Scope

- Subscription payments and billing portal
- Plan upgrades/downgrades
- Fine-grained permission designer / ACL editor
- SSO / enterprise IAM
- Device fleet management for POS hardware

---

## 11. Future Evolution

- Add email + OAuth login methods while keeping `auth_account_id` stable
- Add stronger device trust rules for sensitive actions
- Integrate entitlements snapshot into UC-5 decision inputs (without mixing billing into AuthN)
- Separate Authorization into its own module if complexity grows
