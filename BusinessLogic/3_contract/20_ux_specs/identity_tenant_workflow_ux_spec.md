# UX Spec — Identity, Tenant Selection, and Work Entry

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
   - Archived or removed memberships do not grant access.
   - The UI should never invite the user to “try and fail.”

3. **Reduce friction when there is no choice**
   - Skip selection screens when only one option exists.

---

## 1. Login Screen

### Entry
- Credential: phone number + password (or OTP/reset).
- Login authenticates **Identity only**.

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
- Role label (Owner / Admin / Staff)

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
  - failure → clear reason (capacity reached, suspended, etc.)

---

## 5. Membership State Changes While Logged In

### Scenario: Membership Becomes Archived / Removed

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

## 6. Suspended Membership (Optional UX)

If suspension is supported:

### Option A (Recommended for March)
- Do NOT show suspended memberships at all.

### Option B (Optional Future)
- Show suspended memberships as disabled items.
- Clicking shows:
  > “Your access is temporarily suspended. Contact your manager.”

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
