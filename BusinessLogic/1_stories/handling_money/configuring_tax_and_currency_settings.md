# Configuring Tax & Currency Settings (Owner / Admin)

## Context

Before a POS can be trusted, the owner needs confidence that:
- the tax behavior matches how the business operates
- currency conversion is predictable (USD + KHR is common in Cambodia)
- cash rounding in KHR matches real cash handling at the counter

This is configuration work done by an owner/admin, not something cashiers decide during a rush.

---

## The Situation

An owner/admin is setting up Modula for the first time, or updating settings later because:
- VAT rules changed for the business
- the exchange rate used in-store needs adjustment
- KHR rounding needs to match how the drawer is handled

If the tenant has multiple branches, they may also need:
- the same settings applied across branches, or
- branch-specific differences (rare, but possible)

---

## What the Owner / Admin Expects

They expect:
- to change VAT, FX rate, and KHR rounding intentionally and explicitly
- the change to apply going forward (new sales), without rewriting history
- clear visibility of what the current setting is for a specific branch
- a record of who changed what and when (accountability)

They do **not** expect:
- cashiers to override tax/currency rules during checkout
- old receipts or past sales totals to silently change after a setting update
- to reconcile disputes by reading raw logs

---

## Constraints & Reality

- Some branches may have poor connectivity. Staff may keep operating offline and sync later.
- Owners/admins might change settings outside business hours and expect the next shift to use them.
- Misconfiguration happens (wrong FX rate, wrong VAT), so accountability matters.

The system cannot prevent mistakes, but it must make mistakes **visible and recoverable** without corrupting past truth.

---

## How Modula Helps

Modula supports this by:
- treating VAT/FX/KHR rounding as explicit, branch-scoped configuration
- enforcing these settings at sale finalization time (and snapshotting what was applied)
- keeping past sales/receipts stable even if settings change later
- recording configuration changes for audit and review

---

## What This Story Is Not About

- accounting settlement or tax filing workflows
- dynamic pricing rules or time-based promotions
- complex rule engines

This story is about **operational correctness and trust** for money-related configuration.

