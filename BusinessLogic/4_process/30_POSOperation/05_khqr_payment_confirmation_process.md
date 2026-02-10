# KHQR Payment Confirmation — POS Operation Process (Bakong)

## Process Name
KHQR Payment Confirmation (Generate → Confirm → Allow Finalize)

## Process Type
Cross-Module Business Process (POS Operations + External Payment)

## Status
Defined (March MVP: direct Bakong, KHR+USD, manual finalize)

---

## 1. Purpose

This process defines how Modula treats KHQR as a **real payment rail**:
- QR generation is not the payment.
- A sale is not finalizable until payment is confirmed.

This process intentionally does not define UI layouts. It defines required states and required truth checks.

Related docs:
- `BusinessLogic/4_process/30_POSOperation/10_finalize_sale_orch.md`
- `BusinessLogic/5_modSpec/40_POSOperation/sale_module_patched.md`
- `BusinessLogic/2_domain/40_POSOperation/payment_domain.md`

---

## 2. Preconditions

- Actor is authorized to finalize sales for the current tenant/branch.
- Branch context is resolved (`tenant_id`, `branch_id`).
- Sale draft/cart exists locally (device-scoped).
- An OPEN cash session exists for the branch (Capstone I product rule; still required even for KHQR).
- Branch KHQR receiver configuration exists for:
- selected tender currency (`KHR` or `USD`)
- `(tenant_id, branch_id)`
- Bakong environment configuration is valid (token must match the target environment).
- The Bakong access token is stored and used on the backend only (the POS client must not call Bakong directly).

---

## 3. Inputs (At Generate Time)

- `sale_id` (stable identifier for the draft cart; reused for retries)
- Payable totals (USD, KHR rounded payable) derived from Sale pricing computation
- Tender currency selected for KHQR (`KHR` or `USD`)
- Receiver account id (`toAccountId`) resolved from branch config

Important (March): the cashier must not manually input the KHQR amount.  
KHQR amount is always derived from the computed payable totals.

---

## 4. State Model (Required)

The POS must represent at least these states for KHQR checkout:
- `READY_TO_GENERATE` (amount valid; can generate QR)
- `WAITING_FOR_PAYMENT` (QR generated; awaiting confirmation)
- `PAID_CONFIRMED` (backend confirmed payment proof)
- `EXPIRED` (QR timed out or payment window closed)
- `PENDING_CONFIRMATION` (parked; will be re-checked when connectivity returns)
- `SUPERSEDED` (older attempt replaced by a regenerated attempt)

The sale may only be finalized when state is `PAID_CONFIRMED`.

---

## 5. Generate KHQR (Frontend)

1. Compute payable amount in selected tender currency:
- If `KHR`: use the KHR rounded payable defined by branch policy.
- If `USD`: use the USD payable amount (no KHR rounding applied).
2. Lock the pricing snapshot for this sale intent at QR generation time.
- If the cart changes after generating KHQR, the existing attempt becomes invalid and must be regenerated.
3. Generate KHQR payload string using the KHQR SDK with:
- `toAccountId`
- `currency`
- `amount`
- a unique reference (example: `billNumber` = `sale_id`)
4. Compute `md5` tracking key from the generated KHQR payload (SDK output).
5. Display the QR payload as a QR code and enter `WAITING_FOR_PAYMENT`.
6. Register the payment attempt with the backend (recommended):
- send `(tenant_id, branch_id, sale_id, md5, amount, currency, toAccountId, expires_at)`
- backend stores it for recovery + push updates

Note: registering the payment attempt is not the same as persisting a draft cart. It is a payment artifact.

### 5.1 Regenerate KHQR (March Baseline)

- Regeneration creates a new payment attempt (new KHQR payload and new `md5`).
- When a new attempt is created, the previous attempt becomes `SUPERSEDED`.
- Only the latest attempt is eligible for finalize.

### 5.2 Park Pending Confirmation (March Baseline)

If confirmation cannot be performed (connectivity loss) but the cashier needs to continue:
- allow the cashier to park the attempt as `PENDING_CONFIRMATION` (not finalized)
- the system must provide a way to return to pending attempts later
- when connectivity returns, re-check by `md5` and transition to `PAID_CONFIRMED` if paid

---

## 6. Confirm Payment (Backend)

The backend is the only component that is trusted to confirm payment.

Happy path:
1. Backend calls Bakong `check_transaction_by_md5` using the `md5`.
2. If paid, backend receives proof fields (amount/currency/from/to/hash/time).
3. Backend validates that proof matches expected:
- `toAccountId` equals configured receiver for `(tenant_id, branch_id, currency)`
- `currency` equals selected tender currency
- `amount` equals expected payable in that currency
4. Backend stores confirmation result (linked to `sale_id`) and marks `PAID_CONFIRMED`.
5. Backend pushes a status update to the POS client (Modula-driven "webhook UX").

If unpaid:
- keep `WAITING_FOR_PAYMENT` and allow retry/poll.

If expired:
- set `EXPIRED` and require regeneration.

---

## 7. Manual Finalize (Sale Finalize Entry Point)

When the cashier clicks Finalize:
- the client sends `sale.finalize` with `payment_method = KHQR` and includes `sale_id` + `md5`.
- the backend must re-check the payment proof (or verify a very recent confirmation) before committing sale truth.
- if verification fails, finalize is rejected with no partial writes.

Finalize behavior is owned by:
- `BusinessLogic/4_process/30_POSOperation/10_finalize_sale_orch.md`

---

## 8. Out of Scope (March)

- KHQR refunds/voids
- split payments (cash + KHQR)
- settlement export automation
- cross-device takeover of an in-flight payment without `sale_id`
