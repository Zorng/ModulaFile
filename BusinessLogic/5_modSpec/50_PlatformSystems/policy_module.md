# Policy Module — Core Module (Capstone 1)

**Version:** 1.3  
**Status:** Patched (Branch-scoped policies + cash session requirement is product-enforced)  
**Module Type:** Core Module  
**Depends on:** Authentication & Authorization (Core), Tenant & Branch Context (Core), Audit Logging (Core)  
**Related Modules:** Sale, Inventory, Cash Session, Staff Attendance, Reporting

---


## 1. Purpose

The Policy module centralizes all **business configuration rules** that influence pricing, tax behavior, currency conversion, stock deduction behavior, attendance rules, and cash session permissions.

Policies are **tenant-wide settings** that administrators can configure. These settings control how other modules behave at runtime. Cashiers and managers **do not make policy decisions**; they only operate under rules defined in this module.

The Policy module provides:
- A single place to modify VAT behavior, FX rate, rounding mode  
- Consistent application of inventory subtraction and recipe usage  
- Rules for attendance & shift checking  
- Rules governing cash session behavior  
- Stable values for frontend and backend modules to consume  
- A safe audit trail for configuration changes

Policy values are **read-only at the point of sale** and **cannot be overridden by staff**.

---


## 1.1 Policy Catalog (Canonical Keys)

Policies are **not creatable**. Modula ships a fixed set of policy keys with default values.  
Admins can update them (partial update). Other roles can only read them.

Backend canonical shape (implemented):
- `TenantPolicies` (flattened, tenant-wide)
- Optional category DTOs: `SalesPolicies`, `InventoryPolicies`, `CashSessionPolicies`, `AttendancePolicies`

### Tax & Currency (SalesPolicies)
| UI Label | Data Key | Type | Values / Options | Description |
|---|---|---|---|---|
| Enable VAT | `saleVatEnabled` | boolean | true/false | If on, VAT is applied at checkout. |
| VAT rate (%) | `saleVatRatePercent` | number | 0–100 | VAT percentage used in sale totals. |
| FX rate (KHR per USD) | `saleFxRateKhrPerUsd` | number | > 0 | Conversion rate used to show totals in both USD and KHR. |
| Enable KHR rounding | `saleKhrRoundingEnabled` | boolean | true/false | If on, KHR tender/totals follow rounding rules. |
| KHR rounding mode | `saleKhrRoundingMode` | enum | `NEAREST` \| `UP` \| `DOWN` | How to round KHR. |
| KHR rounding step | `saleKhrRoundingGranularity` | enum | `"100"` \| `"1000"` | Rounding granularity in riel. |

### Inventory Behavior (InventoryPolicies)
| UI Label | Data Key | Type | Values / Options | Description |
|---|---|---|---|---|
| Auto-subtract stock on sale | `inventoryAutoSubtractOnSale` | boolean | true/false | If on, finalize sale triggers inventory deduction via recipe mapping. |
| Enable expiry tracking | `inventoryExpiryTrackingEnabled` | boolean | true/false | If on, restocks can track expiry per batch/lot. |

### Cash Session Control (CashSessionPolicies)
| UI Label | Data Key | Type | Values / Options | Description |
|---|---|---|---|---|
| Require cash session to sell | *(product-enforced)* | boolean | always `true` | **Product rule (Capstone 1):** cash session is required before creating any sale/cart. This is not configurable and is omitted from the Policy API to avoid confusion. |
| Allow paid-out | `cashAllowPaidOut` | boolean | true/false | If on, Paid Out cash movements are allowed in session. |
| Require refund approval | `cashRequireRefundApproval` | boolean | true/false | If on, cash refunds require Manager/Admin approval. |
| Allow manual adjustment | `cashAllowManualAdjustment` | boolean | true/false | If on, manual cash adjustments are permitted (Manager/Admin only). |

### Attendance & Shifts (AttendancePolicies)
| UI Label | Data Key | Type | Values / Options | Description |
|---|---|---|---|---|
| Auto check-in from cash session | `attendanceAutoFromCashSession` | boolean | true/false | If on, starting a cash session also creates a check-in. |
| Require out-of-shift approval | `attendanceRequireOutOfShiftApproval` | boolean | true/false | If on, out-of-shift check-in requires Manager/Admin approval. |
| Enable early check-in buffer | `attendanceEarlyCheckinBufferEnabled` | boolean | true/false | If on, staff may check in before shift start within a buffer. |
| Early check-in buffer (minutes) | `attendanceCheckinBufferMinutes` | number | ≥ 0 | Buffer duration before shift start. |
| Allow manager edits | `attendanceAllowManagerEdits` | boolean | true/false | Reserved for future; in Capstone 1, managers do not edit timestamps (approval only). |

### Update Behavior
- Partial updates use `UpdateTenantPoliciesInput` (all fields optional).
- Writes must be **idempotent** and recorded in Audit Log.
- Reads are cached client-side but refreshed on login and after policy updates.


## 2. Scope & Boundaries

### This module covers:

1. **Tax & Currency Policies**
   - VAT enabled toggle  
   - VAT percentage  
   - Currency conversion using admin-defined FX rate  
   - Rounding mode for Khmer Riel (KHR)

2. **Inventory Behavior Policies**
   - Whether stock should be automatically deducted on sale finalize  
   - Whether recipe deduction is active  
   - (Future) Branch-level overrides

3. **Attendance & Shift Policies**
   - Cash-session-as-attendance (check-in happens when session starts)  
   - Out-of-shift approval requirement  
   - Early check-in buffer (on/off + minutes allowed)

4. **Cash Session Control Policies**
   - Require active cash session to perform sale  
   - Allow Paid Out (on/off)  
   - Cash refund approval requirement  
   - Allow manual cash adjustments (on/off)

### This module does NOT cover:

- Discount rules  
  → handled by **Discount Module** separately  
- Menu pricing or modifiers  
  → Menu module  
- Inventory values or stock counts  
  → Inventory module  
- Shift creation or staff scheduling  
  → Staff Management / Attendance modules  
- Authentication or access control  
  → Auth module  
- Tenant identity (name/logo)  
  → Tenant module  

### Dependencies:

- Auth module (Admin-only updates)  
- Inventory module (uses policies for stock deduction behavior)  
- Sale module (uses VAT, FX, rounding)  
- Attendance module (uses shift rules)  
- Cash Session module (uses session enablement & permissions)

---

## 3. Use Cases (with actor + permission)

---

### UC-1: Admin updates VAT policy

**Actor:** Admin  
**Permissions:** Admin-only  
**Main Flow:**  
1. Admin opens Policy → Tax & Currency  
2. Admin toggles VAT on/off and sets VAT percentage  
3. System saves new settings  
4. Future sales apply new VAT configuration  
5. A policy update log entry is recorded

**Acceptance:**  
- Cashier cannot toggle VAT  
- VAT is applied consistently across sale module and receipt generation

---

### UC-2: Admin updates FX rate and rounding mode

**Actor:** Admin  
**Permissions:** Admin-only  
**Main Flow:**  
1. Admin sets FX rate (KHR per USD)  
2. Admin chooses rounding mode (nearest/up/down)  
3. Admin sets rounding step (e.g., 100៛)  
4. System saves the policy  
5. Future sales convert and round correctly based on new settings

**Acceptance:**  
- Sale UI displays USD exact + KHR rounded  
- KHR tender amount follows rounding policy

---

### UC-3: Admin enables/disables inventory subtraction

**Actor:** Admin  
**Permissions:** Admin-only  
**Main Flow:**  
1. Admin opens Inventory Behavior policy  
2. Admin toggles “Subtract stock on sale”  
3. System updates policy  
4. Inventory deduction logic in Sales follows this setting  
5. Policy update recorded in audit log

**Acceptance:**  
- Turning off subtraction prevents recipe deduction  
- Turning on subtraction causes sale finalize to deduct based on recipe mapping

---

### UC-4: Admin configures attendance behavior

**Actor:** Admin  
**Permissions:** Admin-only  
**Main Flow:**  
1. Admin opens Attendance & Shift policy section  
2. Admin may configure:
   - Cash-session-as-attendance (on/off)
   - Out-of-shift check-in approval (on/off)
   - Early check-in buffer (enable + duration in minutes)
3. System saves policy  
4. Attendance module consumes these values at runtime

**Acceptance:**  
- When cash-session-as-attendance = ON, starting a cash session triggers check-in  
- When out-of-shift approval = ON, cashier requires approval to check-in outside scheduled shift  
- Early check-in buffer allows valid early check-ins

---

### UC-5: Admin configures cash session controls

**Actor:** Admin  
**Permissions:** Admin-only  
**Main Flow:**  
1. Admin opens Cash Session policy  
2. Admin toggles:
   - Require cash session to sell  
   - Allow Paid Out  
   - Require manager/admin approval for cash refunds  
   - Allow manual cash adjustment  
3. System saves the settings  
4. Cash Session module enforces them

**Acceptance:**  
- Sales blocked when “Require cash session” is ON and no active session exists  
- If Paid Out disabled → cashiers cannot issue paid-outs  
- If refund approval required → cashier refund triggers approval flow  
- If manual adjustments disabled → only system-calculated movements allowed

---

## 4. Functional Requirements (FR)

### General Policy Behavior

- **FR-1:** Policies must be stored per tenant.  
- **FR-2:** Only Admin can modify policies.  
- **FR-3:** Managers and Cashiers have read-only visibility.  
- **FR-4:** Policy changes must be persisted atomically.  
- **FR-5:** UI/UX should resemble “phone settings” structure (search + sections).  

### Tax & Currency

- **FR-6:** VAT must be applied automatically based on policy; staff cannot override it.  
- **FR-7:** VAT rate must support decimal percentages (e.g., 10%).  
- **FR-8:** FX rate must be used for USD→KHR conversion in all modules requiring display or tender.  
- **FR-9:** Rounding mode and rounding step must apply when customer pays KHR.  
- **FR-10:** Rounded KHR value must be shown in checkout UI.

### Inventory Behavior

- **FR-11:** If `subtract_stock_on_sale` = true → Inventory module must deduct stock using recipe mapping.  
- **FR-12:** If false → Inventory module performs no deduction.  
- **FR-13:** Policy must integrate with Menu→Inventory recipe mapping.

### Attendance & Shift Rules

- **FR-14:** If cash-session-as-attendance = true → Attendance module must auto check-in when session starts.  
- **FR-15:** If out-of-shift approval = true → Attendance module must request supervisor/admin approval.  
- **FR-16:** If early check-in buffer enabled → Attendance module must validate grace period.

### Cash Session Controls

- **FR-17:** If require-session-to-sell = ON → Sale module must block sale without session.  
- **FR-18:** If allow-paid-out = OFF → Cash Session module must prevent cashiers from initiating Paid Out.  
- **FR-19:** If refund-approval-required = ON → cashier refund triggers an approval workflow.  
- **FR-20:** If manual-adjustment = OFF → no manual adjustments allowed.

### Audit

- **FR-21:** All policy modifications must create an audit entry.  
- **FR-22:** Audit record must contain previous and updated values.  

---

## 5. Acceptance Criteria (AC)

- **AC-1:** Admin can view and edit policies; non-admins cannot.  
- **AC-2:** Policy updates instantly affect dependent modules (Sales, Inventory, Attendance, Cash Session).  
- **AC-3:** VAT, FX, rounding behavior displayed consistently in checkout and receipts.  
- **AC-4:** Inventory deduction follows the toggle state exactly.  
- **AC-5:** Attendance respects shift rules and approval requirement.  
- **AC-6:** Cash Session permissions behave exactly as configured.  
- **AC-7:** All changes produce audit logs visible to admins.  

---

## 6. Out of Scope (Capstone 1)

- Scheduled policies (happy hours, time-based rules)  
- Branch-level policy overrides  
- Multi-VAT, service charges, environmental fees  
- Complex rule engine  
- Policy versioning and rollback  
- Policy export/import  

---

## 7. Developer Notes

- Policies should be cached on frontend and refreshed when navigating to POS.  
- Backend should supply **single consolidated policy object** for simplicity.  
- Do not embed policy logic into Sale, Inventory, Attendance, or Cash Session modules — they must consume policy values from this module.  
- Changes must be backward-compatible for receipts (snapshots handled by Sale module).

## Audit Events Emitted

The following events MUST be written to the Audit Log when triggered (with tenant_id, branch_id where applicable, actor_id, and relevant entity IDs):

# Policy Module — Core Module (Capstone 1)

**Version:** 1.3  
**Status:** Patched (Branch-scoped policies + cash session requirement is product-enforced)  
**Module Type:** Core Module  
**Depends on:** Authentication & Authorization (Core), Tenant & Branch Context (Core), Audit Logging (Core)  
**Related Modules:** Sale, Inventory, Cash Session, Staff Attendance, Reporting  

---

## 1. Purpose

The Policy module centralizes all **business configuration rules** that influence pricing, tax behavior, currency conversion, rounding behavior, inventory subtraction behavior, attendance rules, and cash session permissions.

Policies are **predefined settings** shipped by Modula with default values. Admins can modify values to match store operations. Other roles operate under these rules.

**Scope clarification (patched):** Policies are **branch-scoped**.  
- A tenant may have multiple branches, and **each branch has its own policy values**.  
- All runtime policy resolution in dependent modules must be done using `branch_id`.  
- UI follows an iOS Settings style; when a tenant has multiple branches, policy screens must be opened in a **selected branch context**.

---

## 1.1 Policy Catalog (Canonical Keys)

Policies are **not creatable**. Modula ships a fixed set of policy keys with default values.  
Admins can update them (partial update). Other roles can only read them.

**Canonical keys (unchanged) — scope changed to branch:**
- `saleVatEnabled`, `saleVatRatePercent`, `saleFxRateKhrPerUsd`, `saleKhrRoundingEnabled`, `saleKhrRoundingMode`, `saleKhrRoundingGranularity`
- `inventoryAutoSubtractOnSale`, `inventoryExpiryTrackingEnabled`
- `cashAllowPaidOut`, `cashRequireRefundApproval`, `cashAllowManualAdjustment`
- `attendanceAutoFromCashSession`, `attendanceRequireOutOfShiftApproval`, `attendanceEarlyCheckinBufferEnabled`, `attendanceCheckinBufferMinutes`, `attendanceAllowManagerEdits`

**Deprecated / locked keys (product-enforced):**
- `cashRequireSessionForSales` is treated as **always true** in Capstone 1 and is no longer configurable.

> Note: backend code currently exposes `TenantPolicies` interfaces. After this patch, backend should either:
> - **Option A (preferred):** introduce a `BranchPolicies` storage + DTO, still using the same keys, or
> - **Option B (transition):** keep tenant-wide storage but add “branch override” resolution (not recommended because it conflicts with frontend branch-scoped behavior).

---

## 2. Scope & Boundaries

### This module covers
1. **Tax & Currency Policies (per branch)**
   - VAT enabled toggle
   - VAT percentage
   - FX rate (KHR per USD)
   - Rounding enable/mode/granularity

2. **Inventory Behavior Policies (per branch)**
   - Whether stock should be automatically deducted on sale finalize
   - Whether expiry tracking is enabled

3. **Attendance & Shift Policies (per branch)**
   - Whether cash-session auto check-in is enabled
   - Whether out-of-shift approval is required
   - Early check-in buffer settings

4. **Cash Session Control Policies (per branch)**
   - Require cash session before sale/cart creation
   - Allow paid-out
   - Require refund approval
   - Allow manual adjustment

### This module does NOT cover
- Discounts → Discount module  
- Receipt header fields (logo/name/address/contact) → Tenant & Branch module (data source), Receipt module consumes  
- Menu pricing/modifiers → Menu module  
- Inventory quantities/journals → Inventory module  
- Shift creation/scheduling → Staff Management / Attendance modules  
- Auth/session/roles → Auth module  

---

## 3. Policy Resolution Rule (Branch-scoped)

All policy reads must follow:

- **Inputs:** `(tenant_id, branch_id)`
- **Output:** policy object used by the requesting module

### Branch freeze rule
- **Branch freeze does not change policy values.**
- Freeze blocks operational actions in dependent modules (e.g., cannot sell), but policy remains readable for auditability and reporting.
- Policy UI may be read-only when branch is frozen (implementation choice), but must remain visible.

---

## 4. Policy Catalog Tables (UI)

### Tax & Currency
| UI Label | Data Key | Type | Values / Options | Description |
|---|---|---|---|---|
| Enable VAT | `saleVatEnabled` | boolean | true/false | If on, VAT is applied at checkout. |
| VAT rate (%) | `saleVatRatePercent` | number | 0–100 | VAT percentage used in sale totals. |
| FX rate (KHR per USD) | `saleFxRateKhrPerUsd` | number | > 0 | Conversion rate used to show totals in both USD and KHR. |
| Enable KHR rounding | `saleKhrRoundingEnabled` | boolean | true/false | If on, KHR tender/totals follow rounding rules. |
| KHR rounding mode | `saleKhrRoundingMode` | enum | `NEAREST` \| `UP` \| `DOWN` | How to round KHR. |
| KHR rounding step | `saleKhrRoundingGranularity` | enum | `"100"` \| `"1000"` | Rounding granularity in riel. |

### Inventory Behavior
| UI Label | Data Key | Type | Values / Options | Description |
|---|---|---|---|---|
| Auto-subtract stock on sale | `inventoryAutoSubtractOnSale` | boolean | true/false | If on, finalize sale triggers inventory deduction (via menu-stock mapping / recipe mapping). |
| Enable expiry tracking | `inventoryExpiryTrackingEnabled` | boolean | true/false | If on, restocks track expiry per batch/lot. |

### Cash Session Control
| UI Label | Data Key | Type | Values / Options | Description |
|---|---|---|---|---|
| Require cash session to sell | *(product-enforced)* | boolean | always `true` | Cash session is required before cart/sale creation (Capstone 1). |
| Allow paid-out | `cashAllowPaidOut` | boolean | true/false | If on, Paid Out movements are allowed. |
| Require refund approval | `cashRequireRefundApproval` | boolean | true/false | If on, cash refunds require Manager/Admin approval. |
| Allow manual adjustment | `cashAllowManualAdjustment` | boolean | true/false | If on, manual cash adjustments are allowed (Manager/Admin only). |

### Attendance & Shifts
| UI Label | Data Key | Type | Values / Options | Description |
|---|---|---|---|---|
| Auto check-in from cash session | `attendanceAutoFromCashSession` | boolean | true/false | If on, starting a cash session also creates a check-in. |
| Require out-of-shift approval | `attendanceRequireOutOfShiftApproval` | boolean | true/false | If on, out-of-shift check-in requires approval. |
| Enable early check-in buffer | `attendanceEarlyCheckinBufferEnabled` | boolean | true/false | If on, staff may check in before shift within buffer. |
| Early check-in buffer (minutes) | `attendanceCheckinBufferMinutes` | number | ≥ 0 | Buffer duration before shift start. |
| Allow manager edits | `attendanceAllowManagerEdits` | boolean | true/false | Reserved for future; in Capstone 1, managers do not edit timestamps (approval only). |

### Update behavior
- Updates are partial (only changed fields sent).
- Writes must be idempotent and audited.
- Reads should be cached client-side per branch, refreshed on:
  - login
  - branch switch
  - after a policy update

---

## 5. Use Cases (with actor + permission)

### UC-1: Admin views policies for a branch
**Actor:** Admin  
**Permission:** Admin-only (write), read allowed for Manager/Cashier (optional)  
**Preconditions:** Admin is operating inside a selected `branch_id`  
**Main Flow:**
1. Admin opens Tenant Admin Portal → Settings → Policies
2. Admin selects a branch (if tenant has multiple branches)
3. System displays branch-scoped policy sections and values
**Postconditions:** none

**Acceptance:**
- Branch selection affects displayed values
- Frozen branch policies remain visible

---

### UC-2: Admin updates VAT policy (branch-scoped)
**Actor:** Admin  
**Permissions:** Admin-only  
**Preconditions:** Admin selected branch context  
**Main Flow:**
1. Admin opens Policies → Tax & Currency
2. Admin toggles VAT and sets VAT percentage
3. System saves new policy for that branch
4. Future sales in that branch apply updated VAT
5. Audit log is written
**Postconditions:** branch policy updated

**Acceptance:**
- Cashier/Manager cannot modify VAT
- VAT applies only to the selected branch

---

### UC-3: Admin updates FX rate and rounding (branch-scoped)
**Actor:** Admin  
**Permissions:** Admin-only  
**Preconditions:** Branch selected  
**Main Flow:**
1. Admin sets FX rate (KHR per USD)
2. Admin configures rounding enabled/mode/granularity
3. System saves policy for that branch and audits the change
**Acceptance:**
- Checkout shows USD exact + KHR rounded according to branch policy
- KHR cash payment follows rounding policy

---

### UC-4: Admin configures inventory behavior (branch-scoped)
**Actor:** Admin  
**Permissions:** Admin-only  
**Preconditions:** Branch selected  
**Main Flow:**
1. Admin opens Policies → Inventory Behavior
2. Admin toggles auto subtract on sale and expiry tracking
3. System saves and audits the change
**Acceptance:**
- Sale finalize deducts stock only if branch policy enables it
- Expiry tracking UI is enabled/disabled accordingly

---

### UC-5: Admin configures attendance behavior (branch-scoped)
**Actor:** Admin  
**Permissions:** Admin-only  
**Preconditions:** Branch selected  
**Main Flow:**
1. Admin configures:
   - auto-from-cash-session
   - out-of-shift approval
   - early check-in buffer
2. System saves and audits changes
**Acceptance:**
- Attendance rules enforce branch-specific behavior
- Approval requirement applies within the same branch context

---

### UC-6: Admin configures cash session controls (branch-scoped)
**Actor:** Admin  
**Permissions:** Admin-only  
**Preconditions:** Branch selected  
**Main Flow:**
1. Admin toggles cash session requirements and permissions
2. System saves and audits changes
**Acceptance:**
- Sale/cart creation is blocked until a session starts (Capstone 1 product rule)
- Paid-out/refund/adjustment permissions follow branch settings

---

## 6. Functional Requirements

### General
- **FR-1:** Policies must be stored and resolved per `(tenant_id, branch_id)`.
- **FR-2:** Only Admin can modify policy values.
- **FR-3:** Policy changes must be persisted atomically and audited.
- **FR-4:** UI uses settings-style grouping and provides search.

### Tax & Currency
- **FR-5:** VAT must be applied automatically based on branch policy.
- **FR-6:** FX rate must be used consistently in sale calculations and display.
- **FR-7:** Rounding settings must apply when customer pays in KHR.

### Inventory
- **FR-8:** Auto-subtract on sale must be controlled by branch policy.
- **FR-9:** Expiry tracking enablement must be controlled by branch policy.

### Attendance
- **FR-10:** Attendance enforcement must use branch policy and branch context.
- **FR-11:** Out-of-shift approvals must be requested/approved under correct branch.

### Cash Session
- **FR-12:** Cash session enforcement must use branch context and product rules (Capstone 1 requires sessions for sales).
- **FR-13:** Paid-out/refund/adjustment availability must follow branch policy.

### Branch freeze interaction
- **FR-14:** Branch freeze does not alter policy values.
- **FR-15:** Frozen branch policies remain readable for administration and audit.

---

## 7. Acceptance Criteria

- **AC-1:** Policy resolution uses `branch_id` consistently across all consuming modules.
- **AC-2:** Policy updates affect only the selected branch (no unintended cross-branch bleed).
- **AC-3:** Policy changes create audit entries with old and new values.
- **AC-4:** Frozen branch policies remain visible; freeze blocks operational usage but not visibility.

---

## 8. Out of Scope (Capstone 1)

- Scheduled policies (time-based rules)
- Rule engine / complex conditionals
- Policy versioning and rollback UI
- Cross-branch policy inheritance UI (“copy from HQ”)
- Automated policy deployment pipelines

---

## 9. Developer Notes

- Frontend should cache policy per branch and refresh on branch switch.
- Backend should provide a single consolidated policy object per branch for simplicity.
- Dependent modules must **consume** policies; they must not implement “shadow policy logic” independently.
- If backend is still tenant-wide during transition, explicitly document the mismatch and treat it as a known refactor item; do not silently mix scopes.

---

## 10. Audit Events

- `POLICY_UPDATED`
- `POLICY_RESET_TO_DEFAULT`
- `POLICY_UPDATED`
- `POLICY_RESET_TO_DEFAULT`
