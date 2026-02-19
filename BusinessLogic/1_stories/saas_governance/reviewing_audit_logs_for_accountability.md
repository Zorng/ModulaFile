# Reviewing Audit Logs for Accountability (Owner / Admin)

## Context

In a real café, mistakes and disputes happen:
- a cashier voids a sale and a customer questions what happened
- the cash drawer variance is higher than expected
- someone changes tax/currency settings and totals look different
- inventory is adjusted and the on-hand numbers no longer match reality

When this happens, the business needs a reliable answer to:

> Who did what, when, in which branch, and what was the outcome?

---

## The Situation

An owner/admin needs to review activity history because:
- a customer complaint requires evidence
- a manager needs to understand what happened during a shift
- the business suspects misuse (excessive voids, unusual adjustments)
- a configuration change caused operational confusion

They need this without relying on:
- memory
- screenshots
- “he said / she said”

---

## What the Owner / Admin Expects

They expect:
- an immutable activity history for business-critical actions
- enough context to understand what happened (actor, time, branch, entity IDs, outcome)
- the ability to filter/search by time window, branch, actor, and action type

They do **not** expect:
- a dashboard of KPIs (that is reporting/analytics)
- a notification system (that is operational notifications)
- raw technical stack traces or debugging logs

---

## Constraints & Reality

- Connectivity can be unstable; some actions sync later.
- Staff can leave the business (membership revoked), but history must remain traceable.
- Multi-branch tenants need clarity on “where” an action occurred.

The system cannot prevent every mistake, but it must preserve trustworthy evidence.

---

## How Modula Helps

Modula supports this by:
- writing audit records for state-changing business actions (and important rejections)
- keeping audit records immutable once stored
- enforcing strict tenant isolation and privileged read access (owner/admin)

Audit logs support accountability and dispute resolution. They are not a substitute for reporting.
