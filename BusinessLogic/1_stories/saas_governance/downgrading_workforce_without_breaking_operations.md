# Downgrading Workforce Without Breaking Operations

## Context

Some businesses operate as a team for a period of time, then later decide:
- “We no longer need staff accounts and attendance.”
- “We want to reduce cost and operate as a solo branch for now.”

This decision is commercial, but it affects daily workflows.

---

## The Situation

The owner wants to disable Workforce for a paid branch.

At the time of the downgrade request:
- staff accounts may already exist
- staff may be actively working
- a cash session may be open

The owner wants to reduce cost, but they do not want to “brick the counter” during service.

---

## What the Owner Expects

They expect:
- the downgrade to take effect at the next renewal (end of the already-paid period)
- a clear “pending downgrade” status right away
- the system to prevent new staff sessions from starting while the downgrade is pending
- existing sessions to be able to end cleanly

They do **not** expect:
- staff to suddenly be locked out mid-transaction
- attendance and staff history to be deleted

---

## Constraints & Reality

- Disabling Workforce changes the branch operating model from “team” to “solo operator”.
- The system must avoid unsafe mid-operation cutoffs.
- After Workforce becomes OFF, staff must no longer be able to operate that branch as separate actors.

---

## How Modula Helps

Modula supports this situation by:
- entering a clear `DOWNGRADE_PENDING` state immediately
- blocking new staff check-ins and new staff cash sessions while pending
- making the downgrade effective only at a safe boundary (no active staff attendance; no open staff cash sessions)
- once Workforce is OFF, auto-suspending team operation for that branch (staff can no longer operate that branch)
- keeping historical staff/attendance records viewable (read-only)

---

## What This Story Is Not About

- The internal data model for membership or branch assignment
- HR policy decisions about termination/offboarding

This story is about operational safety:
> “Downgrade should be smooth, not disruptive.”

