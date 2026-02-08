# Void Order — Cross-Module Process (Supporting View)

## Overview
This document is a supporting view of the **order-side effects** of voiding a sale.

**Entry point:** `20_void_sale_orch.md`

## Trigger
Manager/Admin approves & executes a void (sale-side orchestration).

## Domains
- Order
- Inventory
- Cash Session

## Steps
This process occurs as part of the void sale orchestration:

1. Validate void approval and lock execution (Sale/Order)
2. Reverse inventory effects: `22_void_sale_inventory_reversal_process.md`
3. Reverse cash effects: `23_void_sale_cash_reversal_process.md`
4. Mark order `VOIDED` (terminal)

## Idempotency
Void execution is idempotent per `(branch_id, sale_id)` (the orchestration anchor). Order updates must be consistent under retries.
