# UX Spec — Identity, Workspace Entry, and Work Entry

## Metadata
- **Contract Type**: UX Spec
- **Scope**: Login → account layer → tenant layer → branch entry → work entry + loss of access
- **Primary Audience**: Frontend, Backend, QA
- **Owner(s)**: Authentication, Tenant Membership, Staff Profile & Assignment, Access Control, Work Start/End Orchestration
- **Last Updated**: 2026-03-14
- **Delivery Level**:
  - **March**: baseline behavior
  - **Later**: explicitly deferred
- **Related Contracts**:
  - `BusinessLogic/3_contract/10_edgecases/identity_hr_edge_case_sweep_patched.md`
  - `BusinessLogic/3_contract/10_edgecases/identity_hr_pos_boundary_edge_cases_patched.md`
- **Related Processes**:
  - `BusinessLogic/4_process/20_IdentityAccess/10_identity_activation_recovery_orchestration.md`
  - `BusinessLogic/4_process/10_WorkForce/10_work_start_end_orchestration.md`

---

## Purpose

This document defines the **UX behavior and rules** for:
- login
- account-layer workspace entry
- tenant-layer workspace entry
- branch-layer workspace entry
- loss of access

It translates Identity & HR edge-case decisions into **clear UI states**, so frontend and backend stay aligned.

This spec assumes all **domain + edge-case decisions are already locked**.

---

## Core UX Principles

1. **Login is global, work is contextual**
   - Users log into Modula as a person.
   - They then choose *where* they are working.

2. **Only ACTIVE memberships are actionable**
   - Disabled or archived memberships do not grant access.
   - INVITED memberships are shown only in the invitation inbox (not in tenant list).
   - The UI should never invite the user to “try and fail.”

3. **Reduce friction when there is no choice**
   - Skip selection screens when only one option exists.

4. **Management and operations are not the same workspace**
   - Management actions happen in **tenant layer**.
   - Operational actions happen in **branch layer**.
   - Branch-scoped management still requires an explicit branch target, but does not require “entering the branch” first.

---

## Workspace Layer Model

Modula uses three user-facing workspace layers:

### Layer 1 — Account Layer
- The user has authenticated successfully.
- No tenant is assumed yet.
- The user sees:
  - invitation inbox
  - tenants they may enter through ACTIVE memberships
  - create business action

### Layer 2 — Tenant Layer
- The user has selected a tenant (or it was auto-selected).
- This is the workspace for tenant-scoped management and branch entry.
- Role shapes what is visible:
  - `OWNER` / `ADMIN`: management surfaces + branch entry
  - operational staff roles: branch entry only

Small-screen presentation note:
- tenant layer may be presented as a **tenant portal**
- this is a UI shell/presentation choice, not a new business layer

### Layer 3 — Branch Layer
- The user has entered a branch workspace.
- This is the operational workspace for:
  - attendance check-in / check-out
  - cash session open / close
  - sales / checkout
  - viewing effective branch policy / branch discount context

Small-screen presentation note:
- branch layer may be presented as a **branch portal**
- this is a UI shell/presentation choice, not a new business layer

**Important rule:**
- Branch selection is required for **branch-layer operational work**.
- Branch selection is **not** a mandatory universal step immediately after login.

---

## 1. Login Screen

### Entry
- Primary credential: phone number + password.
- Self-service recovery/activation: SMS OTP + set password.
- Login authenticates **Identity only**.

### Account Registration (Self-Service)
If the user does not have an account yet:
1. User submits registration form:
   - first name
   - last name
   - gender (optional)
   - DOB (optional)
   - phone number
   - password (must meet security policy, e.g., length >= 8)
2. System sends OTP.
3. User submits OTP.
4. System verifies OTP and activates the account.
5. User is signed in and proceeds to the **Account layer**.

### Provisioned Staff (No Password Yet)
Staff may be provisioned by a tenant admin before they ever log in.

UX rule:
- Admins/owners never set or know staff passwords.
- If the user does not have a password yet (or forgot it), the UI must support **Activate/Reset via OTP**.

Expected flow:
1. User enters phone number.
2. User selects `Activate / Forgot password`.
3. User sets a new password (must meet policy).
4. System sends OTP and verifies phone ownership.
5. User is signed in (or returned to login with password set).

### Success Outcome
- User is authenticated as an Identity.
- User lands in the **Account layer**.
- No tenant context is assumed yet.

---

## 2. Account Layer

After login or account activation, the user enters the **Account layer**.

This is the workspace where the user:
- sees accessible tenants
- opens invitation inbox
- creates a new business

### Tenant Entry List

### When This Screen Appears
- Identity has **more than one ACTIVE tenant membership**.

### When This Screen Is Skipped
- Identity has exactly **one ACTIVE membership** → auto-select.
- Identity has **zero ACTIVE memberships** → show empty state.

---

### Tenant List Rules

Each list item shows:
- Tenant name
- Optional logo
- Role label (derived from membership facts: `membership_kind` + `role_key`; examples: Owner / Admin / Manager / Cashier / Inventory Clerk)

**Only memberships with status = ACTIVE are shown.**

### Create New Business (Tenant)

This screen must allow the user to create a new business workspace:
- Action: `Create Business`
- On success: the new tenant appears in the list immediately.
- If the new tenant has zero branches, continue into the "first branch activation" flow (see Branch rules below).

---

### Empty State: No Active Memberships

Message example:
> “You are not currently active in any business.”

Actions:
- Create business (start a new tenant)
- Log out
- If you are joining an employer: open **Invitation Inbox** to accept/reject invites, or contact the tenant owner/admin if none exist.

---

## 2.5 Invitation Inbox

### Purpose
Provide a place for users to **accept or reject** pending tenant invites.

### When It Appears
- From account layer (when no ACTIVE memberships exist)
- As an explicit entry in account layer (if invites exist)

### Rules
- Each invite shows tenant name + role label.
- Accept → membership becomes ACTIVE; re-run account layer tenant entry resolution.
- Reject → membership remains inactive; invite is removed or marked rejected.

---

## 3. Tenant Layer

After a tenant is selected (or auto-selected), the user enters the **Tenant layer**.

This is the workspace where the user:
- sees what they can do inside the selected tenant
- enters a branch for operational work
- if authorized, performs tenant management actions

### 3.1 Tenant Layer Behavior by Role

#### Owner / Admin
Must be able to access:
- tenant information
- branch information
- staff management (membership, shift, attendance administration)
- inventory management
- menu management
- discount management
- branch policy/configuration management
- branch entry

Management actions in this layer follow this rule:
- tenant context is active
- if the action affects a branch-scoped resource, the UI must require an explicit target branch (or branch set)

#### Staff / Operational Roles
Must be able to access:
- branch entry list for branches they are assigned to

They do **not** receive tenant-management surfaces by default.

### 3.2 Special Case: Tenant Has Zero Branches

If the selected tenant has **zero branches**:
- Owners/Admins should see a clear call-to-action to **activate the first branch** (payment -> branch is provisioned after confirmation).
- Non-owners should see a clear message:
  > “This business has not activated any branches yet. Contact the owner/admin.”

### 3.3 Branch Entry List

Tenant layer must provide a branch entry list derived from explicit branch assignment.
- Only branches where the user has ACTIVE assignment are shown.
- Optional display of branch location.

---

## 4. Branch Entry and Branch Selection

Branch selection happens when the user attempts to enter **branch-layer operational work**.

It is not a universal post-login requirement.

### When Branch Selection Appears
- Selected tenant has multiple eligible branches for the current user
- AND the user is entering branch-layer operations

### When Branch Selection Is Skipped
- Only one eligible branch exists → auto-select
- No eligible branches → remain in tenant layer and show no-branch-access state

### Branch Selection Rules
- Only branches where the user has ACTIVE assignment are selectable.
- FROZEN branches must not be offered for operational entry.
- If the user changes tenant, previously selected branch context must not be reused across tenants.

---

## 5. Starting Work / Entering Branch Layer

After branch entry is selected:
- System attempts **START_WORK orchestration**.
- UI shows:
  - success → enters branch layer / POS workspace
  - failure → clear reason (capacity reached, disabled, etc.)

---

## 6. Membership State Changes While Logged In

### Scenario: Membership Becomes Disabled / Archived

Trigger:
- Access Control check fails due to membership state change.

UX behavior:
1. Immediately block the action.
2. Show message:
   > “Your access to this business is no longer active.”
3. Redirect user back to:
   - Account layer (if other memberships exist)
   - Login screen otherwise

---

## 7. Disabled Membership (Optional UX)

If disabled memberships are supported:

### Option A (Recommended for March)
- Do NOT show disabled memberships at all.

### Option B (Optional Future)
- Show disabled memberships as disabled items.
- Clicking shows:
  > “Your access is temporarily disabled. Contact your manager.”

---

## 8. Security & Consistency Rules

- Account-layer tenant list is **not cached forever**.
- Membership validity must be re-checked:
  - on login
  - on account-layer tenant entry
  - on tenant-layer management actions where authorization matters
  - on critical actions (START_WORK, END_WORK)

- UI must assume membership can change at any time.
- Branch eligibility must be re-checked when entering branch-layer work.

---

## 9. Explicit Non-Goals

This UX does NOT:
- explain why access was removed in detail
- handle disputes or appeals
- manage invitation reminders or onboarding emails

Those are organizational processes, not UI responsibilities.

---

## Summary

This UX spec ensures:
- the account / tenant / branch workspace model is explicit
- staff can safely work across multiple businesses
- management and operations are separated cleanly
- archived staff disappear cleanly
- no tenant can lock a person out of future work
- frontend behavior stays predictable under edge cases

_End of UX Spec_
