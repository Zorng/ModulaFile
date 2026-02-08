# auth_module.md

## Module: Authentication & Authorization (Core Module)

**Version:** 1.4  
**Status:** Revised (Supports provisioned account activation; aligns credential ownership with Staff onboarding; preserves multi-tenant per account)  
**Module Type:** Core Module  
**Depends on:** Tenant & Branch Context (Core), Audit Logging (Core)  
**Related Modules:** Branch (Core), Staff Management, Policy & Configuration, Sync & Offline, Cash Session, Sale, Staff Attendance

---

## 1. Purpose

The Authentication & Authorization module provides secure access control for Modula by managing:
- **Account authentication** (phone + password, token-based sessions)
- **Tenant membership authorization** (which tenants an account can access)
- **Role-based permissions** within a tenant (Admin / Manager / Cashier)
- **Branch context** resolution (default/selected branch for the session)

This module is phone-first (no email) and supports **multi-tenant per account**.

> Note on ownership: **Auth does not create tenants or branches.** Tenants and branches are **system-created** (e.g., via billing/subscription workflows; temporarily via developer/admin tooling during Capstone I). Auth only controls identity, membership, and access.

---

## 2. Scope (Capstone I)

### Included
- Account registration/activation (phone verification via SMS OTP)
- Login (phone + password)
- Token-based session management (access/refresh tokens)
- Logout (single device session)
- Membership-based tenant access (an account may belong to multiple tenants)
- Tenant selection after login (when multiple memberships exist)
- Role-based authorization per tenant
- Branch context selection and scoping (based on branch access granted by membership/staff assignment)
- Password change (logged-in, requires current password)
- Forgot password (SMS OTP recovery flow)

### Excluded (explicit boundaries)
- Tenant creation and lifecycle triggers (belongs to **Tenant module** + **Billing (Capstone II)**; Capstone I uses developer tooling)
- Branch creation (belongs to **Branch module** + **Billing (Capstone II)**; branches are system-created, not user-created)
- Staff invitation creation and lifecycle (belongs to **Staff Management**)
- User profile/name changes UI and identity editing (belongs to **Account Settings** / future module)
- Provider-side admin / back-office auth

---

## 3. Core Concepts

### 3.1 Account
A unique login identity represented by a phone identifier and credentials.

Note:
- An account may be **provisioned** by the system (e.g., staff onboarding) before the user sets a password.
- Credential setup/reset remains owned by Auth (OTP/self-service); tenant admins do not set passwords.

### 3.2 Membership
A link between an Account and a Tenant, defining:
- tenant access
- role within the tenant
- branch access rules (directly or via staff assignment)
- default branch (or last selected branch)

### 3.3 Role & Permission
Role determines what the user can do inside a tenant:
- Admin (tenant-wide)
- Manager (branch-scoped)
- Cashier (branch-scoped, operational scope)

### 3.4 Branch Context
A session-scoped value representing “which branch the user is currently operating in”.
- Branches are **system-created** (initial branch at tenant creation; additional branches from subscription upgrade).
- Auth resolves/validates branch context based on membership/assignment constraints.

---

## 4. Use Cases

### UC-1: Register / Activate Account (Phone-first with SMS OTP)
Actor: Visitor (new user), System

#### Preconditions
- SMS OTP service available (or fallback is defined)

#### Main Flow
1. User enters phone number.
2. System sends SMS OTP to that phone number.
3. User enters OTP.
4. System verifies OTP (valid + not expired + within attempt limits).
5. System ensures the Account exists:
   - If it already exists (provisioned or existing user), mark phone identifier verified (if not already).
   - If it does not exist, create the Account and mark identifier verified.
6. If password is not set yet, user sets password.
7. System signs the user in.

#### Postconditions
- Account exists in ACTIVE status
- A session is created (tokens issued)

---

### UC-2: Login
Actor: Account user

#### Preconditions
- Account exists and is ACTIVE

#### Main Flow
1. User enters phone number + password.
2. System verifies credentials.
3. System issues tokens (access + refresh).
4. System loads memberships.
5. If user has one membership → enter tenant directly.
6. If user has multiple memberships → require tenant selection (UC-3).

#### Postconditions
- Authenticated session established
- Tenant context is resolvable

---

### UC-3: Select Tenant (Multi-tenant per account)
Actor: Account user

#### Preconditions
- User is authenticated
- User has 2+ memberships

#### Main Flow
1. System presents tenant list user belongs to.
2. User selects a tenant.
3. System sets active tenant context for the session.
4. System resolves role + branch context (UC-4).

#### Postconditions
- Session now has an active tenant context

---

### UC-4: Resolve Branch Context
Actor: Account user, System

#### Preconditions
- User has an active tenant context
- Tenant has 1+ branches (system-created)
- User has branch access within that tenant (based on role / assignment)

#### Main Flow
1. System selects default branch using one of:
   - membership default branch
   - last selected branch
   - assigned branch (for cashier/manager)
   - first available branch (fallback)
2. If the user is allowed to operate multiple branches (typically Admin), the system may prompt branch selection.
3. System stores active branch in session context.

#### Postconditions
- All subsequent authorization checks are scoped to tenant + branch (where applicable)

---

### UC-5: Authorize Request (Role-based Authorization)
Actor: System

#### Preconditions
- User is authenticated
- Active tenant context exists for the request
- Role is known via membership

#### Main Flow
1. System checks access token validity.
2. System checks membership exists and is ACTIVE.
3. System checks role-based permission for requested action.
4. System enforces branch scoping:
   - Admin: tenant-wide access; may select branch context
   - Manager: branch-scoped (assigned branch)
   - Cashier: branch-scoped (assigned branch) + operational scope

#### Postconditions
- Request is allowed or rejected
- Authorization decision may be audit logged for sensitive actions

---

### UC-6: Logout
Actor: Account user

#### Preconditions
- User is authenticated

#### Main Flow
1. User triggers logout.
2. System invalidates refresh token (and optionally current access token server-side).
3. Client clears local tokens and cached contexts.

#### Postconditions
- Session terminated on that device

---

### UC-7: Change Password (Logged-in)
Actor: Account user

#### Preconditions
- User is authenticated (valid session token)
- Account is ACTIVE

#### Main Flow
1. User enters current password and new password.
2. System verifies current password.
3. System updates password hash.
4. System invalidates other sessions (recommended) or all sessions (optional).

#### Postconditions
- Password updated
- Session security is preserved (token invalidation applied per rule)

Notes:
- SMS OTP is not required for change password in Capstone I (to minimize cost and friction).
- Recovery flows use SMS OTP instead (UC-8).

---

### UC-8: Forgot Password (SMS OTP Recovery)
Actor: Account user, System

#### Preconditions
- Account exists for the phone number
- SMS OTP service available

#### Main Flow
1. User enters phone number.
2. System sends SMS OTP.
3. User enters OTP.
4. System verifies OTP.
5. User sets a new password.
6. System invalidates all existing sessions for that account.

#### Postconditions
- Password reset completed securely
- Old sessions revoked

---

### UC-9: Refresh Session (Token Refresh)
Actor: System

#### Preconditions
- Refresh token exists and is valid

#### Main Flow
1. Client requests new access token using refresh token.
2. System validates refresh token and issues new access token (and optionally rotates refresh token).

#### Postconditions
- Session continues without re-login

---

## 5. Requirements

- R1: Phone-first authentication using phone + password.
- R2: Registration must verify phone ownership via SMS OTP.
- R3: Password recovery must use SMS OTP and revoke all existing sessions after reset.
- R4: Change password must require a valid session and current password.
- R5: Token-based sessions must support access + refresh tokens.
- R6: One account may belong to multiple tenants (multi-tenant per account).
- R7: Authorization must be based on membership + role, with branch scoping where applicable.
- R8: Auth must NOT create tenants or branches. Tenant/Branch are system-created (Billing in Capstone II; developer tooling in Capstone I).
- R9: Staff invitations and onboarding lifecycle are not authored by Auth; Auth only supports verification/authentication steps once a valid invitation exists.
- R10: Key auth events should be audit logged (login, logout, password reset, membership changes if applicable).

---

## 6. Non-Functional Requirements

### Security
- Passwords must be stored using secure hashing (e.g., bcrypt/Argon2).
- Tokens must be signed and verified on every protected request.
- Basic brute-force protection (rate limiting, lockout rules) should be in place for login.

### Performance
- Token verification must be fast and not significantly degrade request latency.
- Membership lookup must be efficient (indexes on account_id, tenant_id).

### Reliability
- Auth service must be available for all tenant-bound operations.
- If Auth fails, feature module requests should fail fast with clear error responses.

---

## 7. Acceptance Criteria

- AC1: A user can register with phone number only and must pass SMS OTP verification.
- AC2: A registered user can log in with phone + password and receive a session.
- AC3: A user with multiple tenant memberships can select which tenant to enter.
- AC4: Role and branch scoping are enforced for every request server-side.
- AC5: Logged-in users can change password using current password (no OTP required in Capstone I).
- AC6: Forgot password flow uses SMS OTP and revokes all existing sessions.
- AC7: Logout invalidates the session on the current device.
- AC8: Staff invitation creation is not part of Auth module (belongs to Staff Management).
- AC9: Auth does not expose any “create branch” or “create tenant” operations.

---

## Audit Events Emitted

The following events MUST be written to the Audit Log when triggered (with tenant_id, branch_id where applicable, actor_id, and relevant entity IDs):

- `AUTH_LOGIN_SUCCESS`
- `AUTH_LOGIN_FAILED`
- `AUTH_LOGOUT`
- `ACCOUNT_PHONE_VERIFIED`
- `CREDENTIAL_CHANGE_REQUESTED`
- `CREDENTIAL_CHANGED`
- `MEMBERSHIP_CREATED`
- `MEMBERSHIP_ROLE_CHANGED`
- `MEMBERSHIP_REVOKED`
