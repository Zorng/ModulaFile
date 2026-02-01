# Completing a Sale When the System Is Offline

## Context

The café is open, customers are present, and service must continue.

At some point, the system may lose connectivity:
- the internet drops
- the network becomes unstable
- external services are temporarily unreachable

From the customer’s point of view, none of this matters.  
They still want to order, pay, and receive their items.

---

## The Situation

A customer is ready to pay, but the system is offline.

The cashier must decide:
- whether to stop service, or
- whether to continue and trust the system to catch up later

Stopping service is often not an option.  
Customers are already there, and business cannot pause.

---

## What the Cashier Expects

The cashier expects:
- to continue completing sales without changing their routine
- confidence that each sale is recorded once
- assurance that the system will reconcile later without creating duplicates
- clarity that they are not doing something “wrong” by continuing

They do **not** expect:
- to remember transactions manually
- to re-enter sales later from memory
- to explain technical failures to customers

From the cashier’s perspective, offline operation should feel **as normal as possible**.

---

## What the Customer Expects

The customer expects:
- to be charged correctly
- that their order will be honored
- that the transaction feels legitimate, not improvised

They do not care whether the system is online or offline.  
They care that the café can still serve them.

---

## Constraints & Reality

- Connectivity issues are unpredictable
- Offline periods may last minutes or hours
- Multiple sales may happen before the system reconnects

The system must handle this without:
- losing sales
- duplicating records
- changing past transactions

Once a sale is completed, it must stay completed.

---

## How Modula Helps

Modula supports this situation by:
- allowing sales to be completed even when connectivity is unavailable
- recording each sale reliably at the moment it happens
- synchronizing later without altering the original outcome

When the system reconnects:
- previously completed sales are reconciled automatically
- no additional action is required from the cashier
- the recorded history remains consistent

The cashier can focus on serving customers, not managing outages.

---

## What This Story Is Not About

- Explaining how synchronization works internally
- Handling long-term reporting
- Resolving disputes after the fact
- Fixing connectivity problems

Those belong to other layers and stories.

This story is about **continuing business with confidence**, even when the system cannot connect.