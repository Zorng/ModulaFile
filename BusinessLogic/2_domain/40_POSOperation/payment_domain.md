# Payment Domain Model (Tender + External Confirmation) — Modula POS

**Domain Name:** Payment (Tender + External Confirmation)  
**Domain Type:** Core POS Domain (Cross-cutting)  
**Status:** Defined (March MVP: Cash + KHQR only)  
**Scope:** Tenant + Branch operations  
**Depends On:** Tenant, Branch, Sale/Order, Audit Logging  
**Last Updated:** 2026-02

---

## 1. Domain Purpose & Responsibility

The Payment domain defines:
- how tender is represented for a sale,
- which payment methods require external confirmation,
- and the minimum proof required before a sale may be finalized.

This domain exists to prevent "payment is just a label" drift.

---

## 2. Domain Boundary

### Owns
- The tender representation for a finalized sale (payment snapshot).
- The definition of which payment methods are externally confirmed.
- Validation rules for payment proof (amount/currency/receiver binding).

### Does Not Own
- Pricing computation (VAT/FX/rounding, discounts).
- Inventory deduction.
- Cash accountability ledger (Cash Session).
- Refund/return workflows (future).

---

## 3. Ubiquitous Language

| Term | Meaning |
|---|---|
| Tender | The way a customer pays for a sale |
| Payment Method | Cash or KHQR (March) |
| External Confirmation | Payment acceptance confirmed by the payment network |
| Payment Proof | Evidence that a specific amount/currency reached a specific receiver |
| Receiver | The destination account for KHQR funds (branch-scoped config) |

---

## 4. Supported Methods (March)

### 4.1 CASH
- Confirmation is performed by the cashier at the counter.
- A cash movement is recorded in Cash Session on sale finalize.

### 4.2 KHQR (Bakong)
- Confirmation is performed by checking the payment network status.
- No cash movement is recorded in Cash Session on sale finalize.
- Voids/refunds for KHQR are deferred (Capstone I decision: blocked).

---

## 5. Payment Proof (KHQR) — Minimum Requirements

For a KHQR-paid sale to be finalized, the system must validate that:
- `toAccountId` matches the configured receiver for `(tenant_id, branch_id, currency)`,
- `currency` matches the tender currency selected at checkout (`KHR` or `USD`),
- `amount` matches the payable amount in the selected currency.

The proof must be tied to a stable tracking key (example: `md5`) that can be re-checked by the backend.

---

## 6. Branch-Scoped Receiver Configuration (March)

Receiver configuration is branch-scoped:
- each branch may have a different receiving account
- for March, the system supports different receivers per currency

Conceptual shape:
- `(tenant_id, branch_id, currency) -> toAccountId`

This is configuration, not a policy.

---

## 7. Finalized Sale Payment Snapshot (Write Truth)

On finalize, the Sale module must store an immutable payment snapshot.

Minimum fields:
- `payment_method` (`CASH` | `KHQR`)
- `tender_currency` (`KHR` | `USD`)
- `tender_amount` (Money snapshot; equals payable in tender currency)

For CASH (recommended):
- `cash_received_amount`
- `cash_change_amount`

For KHQR (required):
- `khqr_md5` (tracking key used for verification)
- `khqr_hash` (network transaction hash/reference, if available)
- `khqr_to_account_id` (receiver account id used)
- `khqr_confirmed_at` (timestamp of confirmation used for finalize)

---

## 8. Failure Modes (March Baseline)

| Scenario | Handling |
|---|---|
| KHQR unpaid/expired | Sale not finalizable; allow generate again |
| KHQR paid but mismatched (amount/currency/receiver) | Do not finalize; require manager resolution (future) |
| Network unstable during confirmation | Safe to retry confirmation; do not guess |

