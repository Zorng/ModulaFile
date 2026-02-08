# Responding to Operational Notifications (Manager / Admin)

## Context

In a real café, people are not always in the same place.

A cashier might be at the counter.
A manager might be in the kitchen, on the floor, or not on site.

When something sensitive happens (like a void request), the business still needs a way for the right person to notice and respond without the cashier having to chase them manually.

---

## The Situation

During a busy shift:
- a cashier requests a void for a completed sale
- someone closes the cash drawer
- a close shows an unusually large variance

These are moments where:
- the system cannot decide what the “right” human decision is
- but the system can help humans coordinate by making the state change visible

---

## What the Manager / Admin Expects

They expect:
- to be alerted **in the app** when something needs attention (example: a void request waiting for approval)
- to open the detail quickly and make a deliberate decision
- to know the outcome is recorded and visible

They do **not** expect:
- to be assigned a “task” by the system
- escalation timers or forced responsibility handoffs
- the system to approve or reject on their behalf

---

## What the Cashier Expects

They expect:
- to know whether their request was approved or rejected
- to see the decision outcome clearly so they can respond to the customer

They do **not** expect:
- to be able to bypass approval rules
- the system to silently reverse history without approval

---

## Constraints & Reality

- Notifications can be missed, delayed, or seen by multiple people.
- More than one manager/admin may be available to respond.
- Connectivity can be unstable, so some actions synchronize later.

The system cannot guarantee a human will respond quickly, but it can keep the situation **visible and auditable**.

---

## How Modula Helps

Modula supports this situation by:
- emitting in-app operational notifications for important state changes
- showing a per-user inbox with unread awareness
- deep-linking into the relevant detail (sale/void request, cash session closeout)

Notifications inform coordination, but business truth remains correct even if a notification is missed.

---

## What This Story Is Not About

- marketing or announcements
- SMS/email/push delivery
- assigning ownership, claiming work, or escalation policies

This story is about **operational awareness** that helps humans coordinate responsibly.

