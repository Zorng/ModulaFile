# Accepting KHQR Payment at the Counter

## Context

Some customers prefer to pay by scanning a KHQR code instead of handing over cash.

At the counter, the cashier needs the payment to feel:
- fast
- reliable
- unambiguous

The customer needs confidence that the payment went to the cafe and the order will be fulfilled.

---

## The Situation

A customer places an order and chooses KHQR.

The cashier:
- confirms the items and total
- generates a KHQR code for the payable amount
- waits for confirmation that payment is received
- then finalizes the sale

This is different from cash:
- cash is confirmed by the cashier immediately
- KHQR is confirmed by the payment network

---

## What the Cashier Expects

The cashier expects:
- to generate the QR only when the amount is valid
- a clear state that the system is "waiting for payment"
- a clear state that the payment is confirmed (paid)
- to finalize the sale only after payment is confirmed
- to avoid accidentally finalizing twice when the network is unstable

The cashier does not expect:
- to guess whether the customer actually paid
- to manually reconcile "maybe paid" payments during peak hours

---

## What the Customer Expects

The customer expects:
- the QR code to represent exactly the amount shown at checkout
- the payment to go to the correct business (not a personal account)
- proof that payment was accepted

---

## Constraints & Reality

- Internet can be unstable.
- The customer may scan but not complete payment.
- The customer may pay successfully but the cashier device may momentarily lose connection.

The system must avoid creating a "paid but not recorded" situation, or make it recoverable.

---

## What This Story Is Not About

- Refunds/voids for KHQR payments (deferred for Capstone II)
- Settlement and bank reconciliation exports
- Split payments

