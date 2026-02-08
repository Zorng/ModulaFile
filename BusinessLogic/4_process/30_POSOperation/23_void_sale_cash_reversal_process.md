# Cash Refund on Sale Void — Cross-Module Process

## Process Name
Void Sale → Cash Refund

## Process Type
Cross-Module Business Process

## Status
Defined (Baseline for Capstone 1)

---

## 1. Purpose

This process defines how cash movements recorded during sale finalization are **reversed** when a sale is voided.

It exists to ensure:
- cash accountability remains **auditable** (no deletion/editing of history)
- refunds are **idempotent** and safe under retries/offline replay
- “closed cash session means no more mutations” remains true

This process uses an **append-only compensating movement**:
- it does not edit prior cash-in movements
- it appends a cash-out movement that references the sale being voided

---

## 2. Trigger

**Trigger:** A cash sale void is approved & executed  
(Execution of the *Void Sale* business flow)

This process must be called by:
- `BusinessLogic/4_process/30_POSOperation/20_void_sale_orch.md`

---

## 3. Participating Modules & Responsibilities

| Module | Responsibility |
|------|----------------|
| Sale / Order | Provides immutable sale snapshot (amounts, tender, cash_session_id) |
| Cash Session | Appends the refund cash movement and updates expected cash |
| Process Layer | Orchestrates calls and enforces idempotency |
| Audit | Records the refund action and reason |

---

## 4. Preconditions (March Baseline)

- Sale exists and is eligible for void (typically FINALIZED/VOID_PENDING)
- Sale payment method is CASH (Capstone I rule: QR void/refund is blocked)
- Sale has a `cash_session_id` captured at finalize time
- The related cash session is OPEN (void is blocked if the session is CLOSED)
- Branch is ACTIVE (not frozen), unless explicitly allow-listed by policy

---

## 5. Authoritative Rules

### R-1: Refund amount must come from immutable sale snapshot
The refund uses the finalized sale snapshot (cash collected, currency, change rules), not re-computed totals.

### R-2: Refund is compensating, not mutative
The system must not edit or delete prior cash movements.

### R-3: Refund must be recorded in the sale’s own cash session (March)
To keep reconciliation honest, the refund cash movement must be appended to the **same** `cash_session_id` referenced by the sale.

If that session is no longer OPEN, this process must not proceed.

---

## 6. Process Steps (Happy Path)

1. **Load Sale Snapshot (Sale/Order)**
   - Read sale totals and payment snapshot (cash tender + change)
   - Read `branch_id`, `sale_id`, and `cash_session_id`

2. **Validate Cash Session State (Cash Session)**
   - Verify `cash_session_id` exists and is OPEN

3. **Append Refund Cash Movement (Cash Session)**
   - Append a cash movement:
     - movement_type: `REFUND_CASH` (cash-out)
     - amount: from sale snapshot
     - source_reference: `sale_id`
     - idempotency key anchored by `(branch_id, sale_id)`

4. **Audit Log**
   - Record `CASH_REFUND_RECORDED` (or equivalent) with:
     - tenant_id, branch_id, sale_id, cash_session_id, actor_id, reason

5. **Continue Void Orchestration**
   - Inventory reversal + sale/order state transitions are handled by the void orchestration.

---

## 7. Idempotency & Retry Rules

This process must be safe under:
- network retries
- offline sync replay
- duplicate void execution attempts

Rules:
- Refund recording is idempotent per `(branch_id, sale_id)`
- Duplicate attempts must return success without appending a second refund movement

---

## 8. Failure Modes & Handling

### A. Cash session is CLOSED
- Reject void execution for March baseline (do not allow a refund to be recorded into a closed session).
- Expected UX: “This sale belongs to a closed cash session. Use refund workflow (future) or follow manager policy.”

### B. Refund movement append fails
Preferred behavior:
- Void must not complete “partially”.
- Keep sale in VOID_PENDING until cash refund + inventory reversal succeed, then finalize state transition to VOIDED.

---

## 9. Postconditions

After successful completion:
- Cash Session ledger contains exactly one `REFUND_CASH` movement for the sale
- Expected cash decreases accordingly
- Audit trail is complete

---

## 10. Out of Scope

- Refund workflows after day-close / Z-close (Capstone II)
- Payment gateway refunds (QR refunds)
- Partial refunds (line-level)

---

_End of Cash Refund on Sale Void process_

