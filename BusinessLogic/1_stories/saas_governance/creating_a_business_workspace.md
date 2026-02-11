# Creating a Business Workspace (Tenant)

## Context

Not every Modula user has the same intent.

Some users create an account because they want to **run a business** on Modula.
Other users create an account because they want to **work for a business** that already uses Modula.

To operate a POS, Modula needs a clear business workspace boundary:
- a **Tenant** (the business)
- and later, one or more **Branches** (paid operating locations)

This story captures the moment where a user becomes a business owner in Modula by creating a tenant.

---

## The Situation

A new user signs up and logs in successfully.

At this moment:
- they have no active tenant memberships yet
- they cannot operate branch-scoped workflows yet (sale, cash session, attendance)

They want to start using Modula for their business and expect to create their business space themselves.

Later, the same person may want to create another business (a second tenant) without losing access to the first.

---

## What the User Expects

They expect:
- a clear "Create Business" action when they have no business access yet
- to enter minimal information (example: business name)
- to become the owner/admin of that new business immediately
- to be guided to the next step: activating the first paid branch

They do not expect:
- to contact Modula staff just to create a tenant
- to create an unpaid branch that looks usable but is blocked later
- to be forced to understand internal provisioning concepts

---

## Constraints & Reality

- Modula is multi-tenant SaaS: one identity can operate multiple businesses.
- A tenant may exist with **zero branches** until the first paid branch is activated.
- Branch records must not exist until paid (billing invariant).
- Tenant creation must be controlled (anti-abuse limits) and auditable.

---

## How Modula Helps

Modula supports this situation by:
- allowing any authenticated identity to trigger a controlled `ProvisionTenant` flow ("Create Business")
- creating the tenant in `ACTIVE` state and granting the creator `OWNER` membership
- showing the business in tenant selection immediately after creation
- guiding the user into first-branch activation (payment -> branch provisioned)

---

## What This Story Is Not About

- The payment collection UX itself (KHQR Pay Now details)
- Branch activation and billing anchor logic
- Staff onboarding or invitations
- Detailed tenant profile editing (logo/contact fields)

This story is about the business boundary moment:
> "I created my business workspace, now let me activate the first branch and start operating."

