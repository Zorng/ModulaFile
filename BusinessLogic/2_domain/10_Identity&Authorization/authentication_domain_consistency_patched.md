# Authenticationentication Domain Model — Modula POS

## Domain Name
Auth (Identity & Access)

## Domain Type
Platform / Supporting Domain (Identity & Access)

## Status
Draft (Baseline for Capstone 1)

---

## Purpose

The Auth domain provides **identity** and **authentication** for Modula.
Its goal is to answer, reliably and securely:

- **Who is this user?**
- **Have they proven they are who they claim to be (authenticated)?**

Auth does **not** decide what a user can do inside a business workspace.
Authorization (roles/permissions) is contextual and belongs to Tenant/Staff domains and cross-module processes.

---

## Why This Exists (User Reality)

In a real café, “the person who logs in” is not always the same as:
- the person who owns the business
- the person who pays for the software
- the person who can approve sensitive actions
- the person who operates multiple branches or multiple businesses

If a POS treats everything as “one account,” it becomes fragile when real life happens:
- staff leave and access must be revoked immediately
- managers rotate across branches
- one person works in multiple tenants (multiple cafés)
- subscription/billing later introduces capabilities and suspension rules

Modula separates concerns so the system stays explainable:
- **Auth** proves identity and issues sessions
- **Tenant/Staff** decides membership and roles
- **Subscription/Entitlements (future)** decides what the tenant is allowed to use

---

## 1. Domain Boundary & Ownership

### Owns
- AuthenticationAccount lifecycle (create, disable/enable)
- Authentication methods (phone+password, OTP verification, future email/OAuth)
- Session lifecycle (issue, refresh, revoke, expire)
- Security controls (rate limit, lockout, password reset)
- Audit-grade auth events (login, logout, credential changes)

### Explicitly NOT Responsible For
- Tenant creation / provisioning
- Branch creation / provisioning
- Staff management (hiring, termination, invitations)
- Authorization policy design (role matrices, permission evaluation)
- Subscription, billing, entitlements/capabilities
- POS hardware orchestration (printer, cash drawer, etc.)

Auth produces a trustworthy **actor identity** and **authenticated session**. Other modules interpret that identity in context.

---

## 2. Ubiquitous Language

| Term | Meaning |
|---|---|
| AuthenticationAccount | A global identity record (a human) that can authenticate |
| Identifier | A login identifier such as phone number or email (not the identity key) |
| Credential | Proof method (password hash, OTP verification state, OAuth link) |
| Session | An authenticated session (access/refresh tokens + revocation state) |
| Authentication | Verifying identity (login) |
| Authorization | Deciding permissions in a tenant/branch context (owned elsewhere) |
| Tenant Context | The selected tenant/workspace the user is operating in |
| Branch Context | The selected branch/location the user is operating in (optional) |
| Membership | Relationship that grants a user access to a tenant (owned by Tenant Membership) |
| Revocation | Making a session/token invalid immediately |

---

## 3. Core Concepts

## Tenant and Branch Selection UX Contract

Authentication only proves **who** the person is. After login, the UI must establish a working context:
- If the identity has multiple ACTIVE memberships, the user **selects a tenant**.
- If the tenant has multiple eligible branches for the staff, the user **selects a branch**.
- If there is only one eligible option, the UI may auto-select.

Authentication stores the chosen context inside the session, but **does not own** membership validity.
Authorization decisions remain owned by Access Control and must re-check membership on sensitive actions.

---


### 3.1 AuthenticationAccount (Entity)
Represents a single human identity that can authenticate.

Common attributes:
- auth_account_id (stable internal ID)
- primary_identifier (Phase 0: phone number)
- optional_identifier (future: email)
- status: ACTIVE | DISABLED
- created_at, last_login_at

**Key rule:** phone/email can change; the identity key must remain stable.

---

### 3.2 Credential (Value Object / Sub-Entity)
Represents how the AuthenticationAccount proves identity.

Baseline (Capstone 1):
- password credential (hashed)
- OTP verification state (for registration and password reset)

Future (out of scope now):
- email login
- OAuth providers (Google/Apple)
- SSO for enterprise

---

### 3.3 Session (Entity)
Represents an authenticated session.

Baseline (Capstone 1):
- access token (short-lived)
- refresh token (long-lived)
- device-scoped logout (revocation per device session)
- issued_at, expires_at, revoked_at

Notes:
- Session revocation is required for logout and password reset.
- Session refresh must be safe under concurrent refresh calls (token rotation recommended).

---

### 3.4 Tenant Context & Branch Context (Session-Scope Choices)
Auth may assist the client in selecting context:
- Tenant context is needed when a user has multiple memberships.
- Branch context is needed when operating workflows are branch-scoped.

**Important boundary:** contexts are validated against membership rules, but the truth of memberships/assignments is owned elsewhere.

---

## 4. Invariants

- INV-A1: `auth_account_id` is the stable identity key; identifiers (phone/email) must not be used as primary keys.
- INV-A2: A DISABLED AuthenticationAccount cannot authenticate and must have sessions revoked.
- INV-A3: Password reset must revoke existing sessions (security boundary).
- INV-A4: Authentication does not grant authorization; permissions are evaluated in tenant context by other domains.
- INV-A5: Session refresh must not allow duplicated valid sessions for the same refresh token (rotation/single-use semantics).
- INV-A6: Context selection must be validated (cannot select a tenant/branch without access).

---

## 5. Commands (Write Intents)

Self-contained commands:
- RegisterAuthenticationAccount (phone + OTP verification)
- SetPassword / ChangePassword
- Login (phone + password)
- RefreshSession
- Logout (revoke session)
- ForgotPasswordReset (OTP + new password)
- DisableAuthenticationAccount / EnableAuthenticationAccount

Context helper commands (decision-style):
- ResolveTenantContext (when multiple memberships exist)
- ResolveBranchContext (when branch selection is required)

Auth validates and records session/context choices, but does not create tenants/branches.

---

## 6. Domain Events

Auth emits (or at minimum audit-logs) these events:
- AUTH_ACCOUNT_REGISTERED
- AUTH_ACCOUNT_VERIFIED
- AUTH_ACCOUNT_DISABLED / ENABLED
- USER_LOGIN_SUCCEEDED / FAILED
- USER_LOGGED_OUT
- PASSWORD_CHANGED / RESET
- SESSION_REFRESHED
- SESSION_REVOKED

These events do **not** grant permissions. They are security/audit signals and integration hooks.

---

## 7. Read Model

Typical read queries:
- get account by identifier (phone/email)
- get account status
- get current session context (tenant_id, branch_id if selected)
- list accessible tenants for the user (via memberships owned elsewhere)
- list accessible branches for the user in a tenant (via assignments owned elsewhere)

---

## 8. Cross-Module Integration (Contracts)

### Auth → Tenant/Staff
Auth provides:
- stable `auth_account_id`
- authenticated session context (actor identity)
- optional selected tenant/branch context

Tenant/Staff provides:
- membership and role assignment truth
- branch assignment truth (if used)

### Auth ↔ Subscription/Entitlements (Future)
Auth should remain independent of billing logic.

Future capability enforcement typically happens by:
- tenant status (ACTIVE/SUSPENDED) checked in operational workflows
- entitlements snapshot consulted by POS operations

Auth’s responsibility is only to authenticate the actor.

---

## 9. Out of Scope (Baseline)

- Subscription plans, billing, invoicing, payment statuses
- Fine-grained permission systems beyond basic role lookup
- SSO / enterprise IAM
- Advanced device trust and risk scoring
- POS hardware provisioning and device fleet management

---

## 10. Future Evolution

- Add email + OAuth login methods
- Add stronger session policies (device trust, forced re-auth for sensitive actions)
- Add account recovery enhancements (cooldowns, suspicious activity handling)
- Add admin tooling for bulk session revocation (incident response)
- Support multiple identifiers per identity (phone + email) with safe linking
