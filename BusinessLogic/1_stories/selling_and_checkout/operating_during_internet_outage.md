# Operating During Internet Outage (Cashier / Manager)

## Context

In Cambodia, internet outages are normal.

But customers do not stop buying coffee just because the Wi‑Fi is down.

The cashier still needs to:
- take orders
- collect cash
- keep the line moving

The business expects the system to be reliable even when connectivity is not.

---

## The Situation

The café is busy and the connection drops.

The cashier needs to continue:
- opening a cash session
- finalizing sales
- recording attendance start/end

The manager needs confidence that:
- nothing is lost
- nothing is duplicated
- the system will reconcile when the connection returns

---

## What the Cashier Expects

They expect:
- the POS to keep working even if the connection fails
- a clear offline indicator (so they are not guessing)
- operations to be recorded and synced later without extra steps

They do **not** expect:
- to manually re-enter sales later
- the system to freeze or block customer service

---

## What the Manager / Owner Expects

They expect:
- offline operations to sync exactly once when the connection returns
- cash and inventory effects to remain correct (no double deductions)
- clear visibility if any operation fails to sync

They do **not** expect:
- offline mode to bypass business rules
- hidden failures or silent data loss

---

## Constraints & Reality

- Devices may reconnect hours later.
- Multiple devices may be offline at the same time.
- Some cached data (menu, policy) may be stale.

The system cannot prevent every inconsistency, but it must remain **honest and auditable**.

---

## How Modula Helps

Modula supports outages by:
- queuing offline operations locally with stable IDs
- syncing operations in order when online
- enforcing backend integrity (exactly-once, branch frozen checks)
- showing users clear offline/online status and sync state

Offline mode keeps the café running without weakening business integrity.

---

## What This Story Is Not About

- advanced conflict resolution UI
- offline menu editing or policy updates
- analytics or reporting

This story is about **operational continuity with accountability**.

