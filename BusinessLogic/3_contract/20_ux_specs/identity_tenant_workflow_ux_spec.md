# UX Spec — Identity, Tenant Selection, and Work Entry

## Metadata
- **Contract Type**: UX Spec
- **Scope**: Login → tenant selection → branch selection → work entry + loss of access
- **Primary Audience**: Frontend, Backend, QA
- **Owner(s)**: Authentication, Tenant Membership, Staff Profile & Assignment, Access Control, Work Start/End Orchestration
- **Last Updated**: 2026-02-08
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
- tenant selection
- branch selection
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
   - The UI should never invite the user to “try and fail.”

3. **Reduce friction when there is no choice**
   - Skip selection screens when only one option exists.

---

## 1. Login Screen

### Entry
- Primary credential: phone number + password.
- Self-service recovery/activation: SMS OTP + set password.
- Login authenticates **Identity only**.

### Provisioned Staff (No Password Yet)
Staff may be provisioned by a tenant admin before they ever log in.

UX rule:
- Admins/owners never set or know staff passwords.
- If the user does not have a password yet (or forgot it), the UI must support **Activate/Reset via OTP**.

Expected flow:
1. User enters phone number.
2. User selects `Activate / Forgot password`.
3. System sends OTP and verifies phone ownership.
4. User sets a password.
5. User is signed in (or returned to login with password set).

### Success Outcome
- User is authenticated as an Identity.
- No tenant context is assumed yet.

---

## 2. Tenant Selection Screen

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

---

### Empty State: No Active Memberships

Message example:
> “You are not currently active in any business.”

Actions:
- Log out
- Contact employer / tenant owner

---

## 3. Branch Selection Screen

### When This Screen Appears
- Selected tenant has multiple branches
- AND staff is assigned to more than one branch

### When Skipped
- Only one eligible branch exists → auto-select.

---

### Branch List Rules
- Only branches where the staff has assignment.
- Optional display of branch location.

---

## 4. Starting Work

After tenant + branch selection:
- System attempts **START_WORK orchestration**.
- UI shows:
  - success → enters POS
  - failure → clear reason (capacity reached, disabled, etc.)

---

## 5. Membership State Changes While Logged In

### Scenario: Membership Becomes Disabled / Archived

Trigger:
- Access Control check fails due to membership state change.

UX behavior:
1. Immediately block the action.
2. Show message:
   > “Your access to this business is no longer active.”
3. Redirect user back to:
   - Tenant selection (if other memberships exist)
   - Login screen otherwise

---

## 6. Disabled Membership (Optional UX)

If disabled memberships are supported:

### Option A (Recommended for March)
- Do NOT show disabled memberships at all.

### Option B (Optional Future)
- Show disabled memberships as disabled items.
- Clicking shows:
  > “Your access is temporarily disabled. Contact your manager.”

---

## 7. Security & Consistency Rules

- Tenant selection is **not cached forever**.
- Membership validity must be re-checked:
  - on login
  - on tenant selection
  - on critical actions (START_WORK, END_WORK)

- UI must assume membership can change at any time.

---

## 8. Explicit Non-Goals

This UX does NOT:
- explain why access was removed in detail
- handle disputes or appeals
- manage invitations or onboarding emails

Those are organizational processes, not UI responsibilities.

---

## Summary

This UX spec ensures:
- Staff can safely work across multiple businesses.
- Archived staff disappear cleanly.
- No tenant can lock a person out of future work.
- Frontend behavior stays predictable under edge cases.

_End of UX Spec_
