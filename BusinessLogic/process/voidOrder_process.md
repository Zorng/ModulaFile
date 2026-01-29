# Void Order — Cross-Module Process

## Overview
Reverse a finalized order with explicit approval.

## Trigger
Manager approves void

## Domains
- Order
- Inventory
- Cash Session

## Steps
1. Approve void in Order domain
2. Reverse inventory entries
3. Reverse cash movement
4. Mark order VOIDED

## Idempotency
Void operations are replay-safe per order_id
