# Sale Module — Feature Module (Capstone I)

**Version:** 2.4  
**Status:** Patched (Clarifies order lifecycle and draft persistence rules; preserves existing pricing/discount/cash-session integrity rules)  
**Module Type:** Feature Module  

**Depends on:**  
- Auth & Authorization (Core)  
- Tenant & Branch Context (Core)  
- Policy & Configuration (Core)  
- Menu (Feature)  
- Inventory (Feature)  
- Cash Session & Reconciliation (Feature)  s
- Discount Management (Feature)  
- Sync & Offline Support (Core)  
- Receipt (Feature)  
- Audit Logging (Core)  

---

## 1. Purpose

The Sale Module handles day-to-day transaction processing in Modula POS.

It is responsible for:
- building cart/draft orders,
- computing final prices,
- enforcing cash session rules,
- applying taxes and discounts,
- finalizing sales (financial record),
- creating and managing order fulfillment status,
- and persisting immutable transaction records for reporting.

The Sale module is the single source of truth for finalized transaction totals.  
Order status changes must never modify finalized sale totals.

---

## 2. Scope & Boundaries

### Includes
- Menu browsing (always allowed)
- Draft sale (cart) creation and management (local-only)
- Sale types: Dine-in, Takeaway, Delivery
- Payment methods: Cash, KHQR (Bakong)
- Cash tender currency selection (USD or KHR) for cash payments
- Change calculation for cash payments
- KHQR tender currency selection (USD or KHR) for KHQR payments
- KHQR payment confirmation (manual finalize after paid)
- Automatic application of:
  - VAT (from Policy — branch-scoped)
  - Currency conversion & KHR rounding (from Policy — branch-scoped)
  - Discounts (via Discount module, branch-scoped, lock-in on finalize)
- Dual-currency totals display (USD & KHR)
- Finalize sale (creates immutable financial record)
- Order creation after finalize and fulfillment statuses:
  - IN_PREP → READY → DELIVERED
  - VOIDED (via approved void)
  - CANCELLED (controlled; reserved)
- eReceipt viewing aligned with Orders (click an order → open its eReceipt)
- Void request (Cashier) + Void approval/execution (Admin/Manager)
- Sale voiding effects (cash + inventory reversal) within Capstone I rules
- Offline finalize with idempotent sync

### Excludes
- Split bills
- Partial payments
- Loyalty / customer accounts
- Advanced payment settlement
- Refund workflows after day-close (Capstone II)
- Voiding KHQR payments (Capstone I decision: blocked)

---

## 3. Core Concepts

### 3.1 Draft Order (Cart)
A temporary, mutable order state while items are selected.

**Capstone I rule:** Draft orders are **local-only** and are **not persisted to the backend** to avoid unnecessary database writes and to keep the backend focused on immutable, auditable records.

#### 3.1.1 Draft persistence (device-scoped)
Within an OPEN cash session, the draft cart **may persist on the device** (e.g., IndexedDB/local storage) so it can survive:
- accidental reloads,
- browser/app restart,
- brief connectivity loss.

Draft persistence is **scoped** to the current operational context:
- `tenant_id`, `branch_id`, and `cash_session_id`
- `device_id` (or equivalent device-scoped identifier)
- the current cashier identity (`actor_id`)

If the context changes, the system must prompt the user to resolve the unfinished draft (finalize or discard).

#### 3.1.2 Single active draft per device
To prevent “draft explosion” and simplify operator workflows, the system enforces:
- **At most one active draft cart per device per branch per OPEN cash session.**

This rule supports both:
- **single-register workflows** (one primary counter device), and
- **multi-device workflows** (multiple cashiers on their own devices),
while keeping draft management deterministic and local to each device.

### 3.2 Finalized Sale (Financial Record)
An immutable transaction record created at checkout.

Once finalized:
- totals must never change
- discounts must not be recomputed
- historical data must remain stable

Sale status is used to represent financial lifecycle:
- FINALIZED
- VOID_PENDING
- VOIDED

### 3.2.1 Finalized Sale Snapshot (Required for Receipt + Reporting)

On finalize, the Sale module must persist an **immutable sale snapshot** that supports:
- Receipt rendering (customer dispute resolution)
- Management reporting aggregation (sales performance insight)
- Auditability under offline replay/idempotent sync

At minimum, the finalized sale snapshot must include:

Sale header snapshot:
- `tenant_id`, `branch_id`
- `sale_id`
- finalized timestamp
- sale type (dine-in / takeaway / delivery)
- sale status (FINALIZED / VOID_PENDING / VOIDED)
- cashier identity (`actor_id`) (recommended; supports drill-down and accountability)

Totals snapshot (captured at finalize time; no recompute later):
- subtotal
- discount totals + discount snapshot (see 3.5)
- VAT totals (if enabled) + VAT policy snapshot values applied
- FX rate + KHR rounding policy snapshot values applied
- grand totals (USD + KHR) and any rounded payable value used at checkout

Tender snapshot:
- payment method (cash / KHQR; extend later)
- if cash: tender currency (USD vs KHR) and change results (recommended)
- if KHQR: tender currency (USD vs KHR) and KHQR proof fields (required):
  - `khqr_md5` (tracking key used for confirmation)
  - `khqr_hash` (network transaction reference, if available)
  - `khqr_to_account_id` (receiver account id used)
  - `khqr_confirmed_at` (timestamp of confirmation used for finalize)

Line snapshot (for item/category reporting and receipts):
- `menu_item_id`
- menu item display name snapshot
- menu category snapshot (category id + category display name), or null → treated as **Uncategorized**
- quantity
- line monetary snapshot (line totals and optionally per-line discount amounts)

Design rule:
- Reporting must aggregate item/category metrics from the **stored sale line snapshot**.
- Do not join to current Menu in a way that rewrites historical reports when items/categories are renamed, moved, or deleted later.

### 3.3 Order (Fulfillment Record)
An operational record used for fulfillment tracking.
- Created when a sale is finalized.
- Default status: `IN_PREP`
- Status may transition to: `READY`, `DELIVERED`, or `CANCELLED` (controlled)
- When a sale is voided, the associated order transitions to `VOIDED` (terminal).
- Changing order status must not change finalized sale totals.

**Important:** Order status is **fulfillment-only**. Financial reversal is represented by the **Sale status** (`VOID_PENDING` → `VOIDED`) and by reversal entries in **Cash Movement** and **Inventory Journal** (where applicable).

#### 3.3.1 Order lifecycle (state machine)
The Order lifecycle is intentionally simple for Capstone I:
- `IN_PREP` → `READY` → `DELIVERED`
- `IN_PREP` or `READY` → `VOIDED` (only via approved void workflow)
- `CANCELLED` is reserved for controlled administrative termination (not used by Cashier flows in Capstone I).

---

### 3.4 Branch Context and Branch Provisioning (Patched)
The Sale module always operates within an active branch context (`branch_id`).

**Branch provisioning rule (Capstone I):**
- A branch is provisioned by the system, not created by an end-user inside operational modules.
- At initial onboarding, the system provisions Tenant + default Branch (tenant creation currently developer-triggered in Capstone I; billing engine will automate in Capstone II).
- Additional branches are provisioned when the tenant subscribes to add capacity (billing scope in Capstone II; developer-triggered in Capstone I).

Therefore:
- Sale does not include any use case to create a branch.
- Sale assumes `branch_id` is available via the authenticated user’s membership / session context.

---

## 3.5 Discount Resolution, Scope, and Lock-in

### 3.5.1 Branch-scoped resolution (explicit)
All discount resolution is scoped by branch.
- The Sale module MUST resolve discounts using:
  - `tenant_id`
  - current `branch_id`
  - cart contents (menu_item_id + modifiers/add-ons selection)

This means:
- Item discounts apply only if effective for the current branch.
- Branch-wide discounts are resolved for the current branch (not tenant-global unless explicitly configured as such by Discount).

### 3.5.2 Responsibility split
- The Discount module determines which discounts are effective for the current branch and returns them as “effective discount rules”.
- The Sale module:
  - computes discounted totals,
  - applies stacking rules,
  - and stores a discount snapshot on finalize.

### 3.5.3 Stacking rule (locked)
Discounts stack multiplicatively, in a deterministic order:
- Item-level combined discount applies per cart line
- Branch-wide discount applies to the subtotal after line calculations

### 3.5.4 Line discount formula (locked)
Item-level discounts apply to the full cart line base amount:

`line_total = (base item price + add-ons/modifiers) × (1 − combined_item_discount)`

### 3.5.5 Snapshot requirement (locked, branch-aware)
The discount snapshot stored on finalize MUST include:
- `branch_id` used for resolution
- applied discount IDs and names
- applied discount percentages
- the discounted line base amount (including add-ons/modifiers)
- computed discount amounts per line
- branch-wide discount (if any) and its computed amount

This prevents historical sales from changing if discount rules are edited later.

---

## 3.6 Policy Dependencies (Canonical Keys) — Branch-scoped

**Important:** Policies are configured and enforced per branch (branch-scoped).  
Sale computations MUST load policy values using `(tenant_id, branch_id)` context and treat the returned policy snapshot as applying to the current branch only.

The Sale Module must read policy values using the backend-implemented keys (from `TenantPolicies`), but interpreted as branch policy values in the Sale context:

### Tax & Currency
- `saleVatEnabled` (boolean)
- `saleVatRatePercent` (number)
- `saleFxRateKhrPerUsd` (number)
- `saleKhrRoundingEnabled` (boolean)
- `saleKhrRoundingMode` (`NEAREST | UP | DOWN`)
- `saleKhrRoundingGranularity` (`100 | 1000`)

### Cash Session Control
- Cash sessions are required for cart/sales in Capstone 1 (product rule; not configurable).

### Inventory Deduction (Not a Policy)
- Sale-based inventory deduction is derived from **Menu composition** (recipe/direct-stock mapping), not a policy toggle.
- Finalize should produce `deduction_lines` (may be empty). Inventory deduction executes only when `deduction_lines` is non-empty.

---

## 3.7 Cash Session Dependency (UX Rule)

**Product rule (Capstone 1):** a user must have an OPEN cash session in the current branch before creating a cart / adding items / finalizing a sale.

This prevents draft-sale spam, enforces cash discipline, and ensures X/Z reporting is always available.

---

## 3.8 Data Integrity (Locked for Capstone I)

Cash sessions are required for cart/sales, but backend integrity rules still apply.

At minimum, backend must guarantee:
1. **Atomic finalize** (sale + order + cash movement + inventory deduction where applicable)
2. **Idempotency / duplicate prevention** for finalize operations (including offline sync replay)
3. **Concurrency-safe stock mutation** when inventory deduction executes (i.e., when `deduction_lines` is non-empty)
4. **Server-authoritative ordering** for finalize operations

---

## 4. Pricing Computation Rules (Locked for Capstone I)

### 4.1 Cart line calculations
For each cart line:
- Line base amount is the sum of:
  - base menu item price
  - selected modifier/add-on prices
- Effective item discount is resolved by branch context
- Item-level discounts apply to the entire line base amount
- Line total is computed as:  
  `line_total = (base_price + sum(modifier_addons)) × (1 - combined_item_discount)`
- Quantity editing multiplies the final line total accordingly.

### 4.2 Totals sequence (Cart / Pre-checkout)
The system computes (sequence must be preserved):
1. **Subtotal**  
   `subtotal = sum(all line_total)`
2. **Branch-wide discount (branch-scoped)**  
   - resolved for current `branch_id`
   - applied on subtotal
3. **VAT (branch-scoped policy)**  
   - applied after discounts when `saleVatEnabled = true`
   - rate comes from `saleVatRatePercent`
4. **Grand totals**  
   - displayed in both USD and KHR
   - conversion uses `saleFxRateKhrPerUsd`
5. **KHR rounding (branch-scoped policy)**  
   - applied when `saleKhrRoundingEnabled = true`
   - rounding uses:
     - `saleKhrRoundingMode`
     - `saleKhrRoundingGranularity`

### 4.3 Cash tender & change
If payment method is Cash:
- cashier selects cash tender currency: USD or KHR
- system computes amount due in both currencies, but change is computed in the selected tender currency
- KHR tender uses rounding policy and change respects rounding rules

### 4.4 KHQR tender (March)
If payment method is KHQR:
- cashier selects KHQR tender currency: USD or KHR
- KHQR amount is not manually entered; it is derived from the computed payable total in the selected currency
- generating KHQR locks the pricing snapshot for that sale intent; any cart change requires regenerate

---

## 5. Use Cases

### UC-1 — Access Sale Screen
**Actors:** Cashier, Manager, Admin

**Preconditions**
- User authenticated
- User has an active membership granting access to at least one tenant + branch
- Branch context active (`branch_id` resolved)
- Branch exists (provisioned by system; not user-created within Sale)

**Main Flow**
1. User opens their portal
2. Selects “Sale”
3. Sale screen loads (menu browsing enabled)

**Postconditions**
- Sale UI ready
- Cart actions enabled/disabled per policy

---

### UC-2 — Start Draft Order (Add Item to Cart)
**Actors:** Cashier, Manager, Admin

**Preconditions**
- Menu items exist and are assigned to current branch
- Branch context active (`branch_id` resolved)
- Branch policy loaded for current branch
- An OPEN cash session exists for the user at the branch (Capstone 1 product rule)

**Main Flow**
1. User taps menu item
2. Local draft cart is created (or reused if already open)
3. User selects modifiers (if any)
4. Item is added to cart
5. Totals update live

**Postconditions**
- Draft exists locally
- No inventory or cash changes
- No server write

---

### UC-3 — Cart / Pre-checkout (Edit + Compute)
**Actors:** Cashier, Manager, Admin

**Preconditions**
- Draft cart contains items
- Branch context active (`branch_id` resolved)
- Branch policy loaded for current branch

**Main Flow**
1. User opens Cart / Pre-checkout.
2. User may:
   - edit quantity of cart items
   - remove cart items
   - review item-level discounts (applied automatically, branch-scoped)
3. User selects:
   - sale type (dine-in / takeaway / delivery)
   - payment method (cash / KHQR)
   - if cash: tender currency (USD or KHR) and amount received (for change calculation)
   - if KHQR: tender currency (USD or KHR)
4. System computes and displays:
   - line totals (including item discounts applied to base+add-ons)
   - subtotal
   - branch discount effect (resolved for current `branch_id`)
   - VAT effect (controlled by `saleVatEnabled`, `saleVatRatePercent`) — branch-scoped
   - grand totals in USD and KHR (using `saleFxRateKhrPerUsd`) — branch-scoped
   - KHR rounded payable (using `saleKhrRounding*`) — branch-scoped
   - cash change (if cash payment)
5. If payment method is KHQR:
   - system allows KHQR generation for the payable amount in the selected tender currency
   - system does not allow finalize until payment is confirmed (paid) via backend verification

**Postconditions**
- Order is ready for finalize
- No server write yet (until finalize)

---

### UC-4 — Finalize Sale (Checkout)
**Actors:** Cashier, Manager, Admin

**Preconditions**
- Pre-checkout complete
- Branch context active (`branch_id` resolved)
- Branch policy loaded for current branch
- An OPEN cash session exists for the user at the branch (Capstone 1 product rule)

**Main Flow**
1. User confirms checkout.
2. Sale module:
   - computes final totals using current branch policies
   - resolves discounts for current `branch_id`
   - locks discount snapshot (including `branch_id` used)
   - locks policy snapshot values used (VAT/FX/Rounding) as applied for this branch
   - if payment method is KHQR: requires KHQR proof and verifies it via backend confirmation before committing truth
   - persists finalized sale (`status = FINALIZED`)
3. If cash:
   - cash movement is recorded via Cash Session module
4. If sale evaluation produces `deduction_lines`:
   - inventory deduction is executed
5. An Order is created with status IN_PREP
6. eReceipt becomes available for the finalized sale/order

**Postconditions**
- Sale stored as FINALIZED (immutable)
- Order stored as IN_PREP
- Side effects applied once
- Backend integrity guarantees apply (see 3.8)

---

### UC-5 — View Orders & eReceipt
**Actors:** Cashier, Manager, Admin

**Preconditions**
- At least one order exists for current branch context (role-scoped visibility applies)

**Main Flow**
1. User opens Orders list.
2. User clicks/taps an order.
3. System opens eReceipt view for that order’s finalized sale.

**Postconditions**
- None (read-only)

---

### UC-6 — Update Order Fulfillment Status (Progress Only)
**Actors:** Cashier, Manager, Admin (role-scoped per branch context)

**Preconditions**
- Order exists
- Order is not terminal (DELIVERED, VOIDED, or CANCELLED)

**Main Flow**
1. User selects an order.
2. User updates status:
   - IN_PREP → READY
   - READY → DELIVERED
3. System records status transition and logs action.

**Postconditions**
- Order fulfillment status updated
- Finalized sale totals unchanged

**Key rule:** Cashier must not cancel finalized orders. Order termination is handled via approved void (UC-8).

---

### UC-7 — Request Void (Cashier)
**Actors:** Cashier

**Preconditions**
- Sale is FINALIZED
- Sale payment method is eligible for void workflow
- Branch context active

**Main Flow**
1. Cashier opens an order and selects “Request void”.
2. Cashier provides a reason.
3. System creates a Void Request record with status PENDING.
4. System sets Sale status = VOID_PENDING.
5. Orders UI shows a badge “Void requested”.
6. System emits an OperationalNotification to approvers (best-effort, idempotent):
   - ON-01 `VOID_APPROVAL_NEEDED:{branch_id}:{sale_id}`

**Postconditions**
- Void request is recorded (PENDING)
- Sale is VOID_PENDING
- No reversal effects occur yet

---

### UC-8 — Approve & Execute Void (Manager/Admin)
**Actors:** Admin, Manager

**Preconditions**
- Void request exists and is PENDING
- Sale status is VOID_PENDING
- Capstone I business rules:
  - Payment method must be CASH (KHQR void blocked)
  - Related cash session must be OPEN (void blocked if session is CLOSED)

**Main Flow**
1. Manager/Admin opens the order/sale detail and reviews void request.
2. Actor selects “Approve void” (and provides optional review note).
3. System validates business rules.
4. System executes void:
   - Sale status → VOIDED
   - Order status → VOIDED
   - Cash refund movement (`REFUND_CASH`) recorded (via Cash Session module)
   - Inventory reversal executed (if inventory deduction was applied on finalize)
5. System marks void request as APPROVED.
6. System logs action (audit).
7. System emits an OperationalNotification to the requester (best-effort, idempotent):
   - ON-02 `VOID_APPROVED:{branch_id}:{sale_id}`

**Alternative Flow — Reject**
1. Actor selects “Reject void” and provides reason.
2. System marks void request REJECTED.
3. System sets Sale status back to FINALIZED.
4. Audit log recorded.
5. System emits an OperationalNotification to the requester (best-effort, idempotent):
   - ON-03 `VOID_REJECTED:{branch_id}:{sale_id}`

**Postconditions**
- If approved: Sale VOIDED and effects reversed once; Order VOIDED
- If rejected: Sale restored to FINALIZED; Order unchanged

---

### UC-9 — Offline Finalize & Sync
**Actors:** System

**Preconditions**
- Offline mode active
- Draft cart exists

**Main Flow**
1. User finalizes sale while offline.
2. Sale and order are created locally.
3. On reconnect, sync runs idempotently.
4. Server receives and persists exactly-once.

**Postconditions**
- No duplicates
- Final totals preserved

---

## 6. Functional Requirements
- FR-1: Admin, Manager, Cashier can perform sales
- FR-2: Sale requires an OPEN cash session for cart/sales (Capstone 1 product rule)
- FR-3: Discounts are resolved per branch context, stacked multiplicatively, and snapshot locked on finalize
- FR-4: VAT/FX/Rounding policies are loaded and applied per branch using the canonical keys
- FR-5: Cart supports quantity edit and removal prior to finalize
- FR-6: Cash tender supports USD/KHR selection and change computation
- FR-7: Finalized sale creates an order in IN_PREP
- FR-8: Order status updates do not modify finalized sale totals
- FR-9: Inventory deduction executes on finalize only when the sale produces `deduction_lines` (from Menu composition)
- FR-10: Offline finalize is safe and idempotent
- FR-11: Draft carts do not pollute database
- FR-12: Cashier can request void; Manager/Admin can approve/reject
- FR-13: Voiding KHQR sales is blocked in Capstone I
- FR-14: Voiding is blocked if related cash session is CLOSED
- FR-15: Approving void marks the order VOIDED and reverses cash/inventory exactly once
- FR-16: Finalize sale must be idempotent (duplicate prevention on retries and offline replay)
- FR-17: KHQR finalize requires backend-confirmed payment proof bound to amount/currency/receiver

---

## 7. Acceptance Criteria
- AC-1: Menu browsing is always allowed
- AC-2: Cart creation is blocked without an active cash session (Capstone 1 product rule)
- AC-3: Cart allows quantity edits and recomputes totals correctly
- AC-4: Cash tender supports USD/KHR and computes change in tender currency
- AC-5: Discounts apply to (base + add-ons) before discount, are branch-scoped, and are immutable after finalize
- AC-6: VAT/FX/Rounding values applied during pricing match the policies configured for the current branch
- AC-7: Finalize creates an order in IN_PREP
- AC-8: Order status can transition IN_PREP→READY→DELIVERED without changing sale totals
- AC-9: Clicking an order opens its eReceipt
- AC-10: Cashier can submit void request; sale shows VOID_PENDING badge
- AC-11: Manager/Admin can approve void only for CASH sales with OPEN cash session
- AC-12: Approving void sets sale VOIDED, order VOIDED, and reverses effects; audit logged
- AC-13: Rejecting void restores sale FINALIZED and keeps fulfillment unchanged
- AC-14: If the same finalize request is retried (network drop/double submit/offline replay), the backend persists it exactly once
- AC-15: KHQR checkout cannot finalize until payment is confirmed; finalize validates proof matches expected tender

---

## 8. Deletion Rules

| Entity | Rule |
|---|---|
| Draft | Discardable (local) |
| Sale | Not deletable |
| Order | Not deletable (status-based terminal states) |
| Void request | Not deletable (status-based lifecycle) |
| Void | Logical reversal only |

---

## 9. Out of Scope
- Split payments
- Partial refunds
- Loyalty
- Advanced payment settlement
- KHQR refunds/voids
- Voids after day close / Z-close (refund workflows in Capstone II)

---

## Audit Events Emitted

The following events MUST be written to the Audit Log when triggered (with tenant_id, branch_id where applicable, actor_id, and relevant entity IDs):
- CART_CREATED
- CART_UPDATED
- SALE_FINALIZED
- ORDER_STATUS_UPDATED
- VOID_REQUESTED
- VOID_APPROVED
- VOID_REJECTED
