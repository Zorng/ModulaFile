# Revoking Staff Workspace Access (History Preserved)

## Context

Over time, every café’s staff roster changes.

People leave.
New hires join.
Seasonal workers come and go.

If the system only ever adds staff but never “cleans up”, the team list becomes confusing:
- old names stay in dropdowns
- managers waste time choosing the right person
- it becomes unclear who is actually active

At the same time, businesses need history:
- attendance should remain traceable
- past sales should remain attributable
- audits should remain verifiable

So the system must support removing someone from the active roster **without deleting them from history**.

---

## The Situation

A staff member has stopped working for the café and should no longer appear as an active operator.

The owner wants to:
- keep the active staff list clean
- preserve historical records
- avoid future confusion during operations

---

## What the Owner Expects

The owner expects:
- the staff member disappears from active lists used for daily operations
- historical records remain correct and readable
- the system does not “rewrite the past”
- the action is auditable (who revoked access, when)

They do **not** expect:
- to delete records manually
- to lose the ability to explain past attendance or sales
- to break reports or receipts because a staff record vanished

---

## Constraints & Reality

Staff may still have unfinished responsibilities:
- they might still be checked-in
- they might have a cash session still open

The system must avoid a dangerous illusion:
> “This person is gone, so their responsibilities disappeared too.”

Even when someone’s access is revoked, operational truth must remain intact.

---

## How Modula Helps

Modula supports access revocation by:
- revoking tenant membership (and/or branch assignment) to remove workspace access,
- separating “active roster visibility” from “historical existence”
- preserving identity and history for audit and reporting
- ensuring operational responsibilities are handled explicitly (not erased)

This keeps daily operations clean while protecting the integrity of business history.

---

## What This Story Is Not About

- deleting staff history
- rehiring or reactivating policy decisions (this is handled as a new invite/grant of membership)
- payroll/HR systems
- performance evaluation

This story is only about **removing someone from the active roster without deleting history**.
