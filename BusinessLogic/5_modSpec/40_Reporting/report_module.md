# Report Module

**Version:** 1.3  
**Status:** Patched (Demo scope: Cash X + Z reports; access rules aligned to implementation)  
**Module Type:** Feature Module  
**Depends on:**  
- Authentication & Authorization (Core)  
- Tenant & Branch Context (Core)  
- Policy & Configuration (Core) — branch-scoped resolution  
- Sale Module  
- Cash Session Module  
- Audit Logging (Core)

---

## 1. Purpose

The Report Module provides **accurate, auditable business insights** for administrators by aggregating operational data from sales and cash sessions.  
It ensures that **financial truth is preserved**, while clearly exposing provisional states such as pending voids without silently mutating historical records.

This module prioritizes:
- Accounting safety
- Operational transparency
- Auditability
- Clear separation between finalized and provisional data

---

## 2. Scope (Capstone 1)

Included:
- Cash session reporting (X + Z)
- Branch-scoped access control
- Branch-scoped policy interpretation for display only (when applicable)
- **Reporting remains accessible for frozen branches** (read-only history)

Excluded (Capstone 2+):
- Branch-level sales summary reporting
- Branch-level item-level sales reporting
- Cross-branch aggregated reports (tenant-wide totals)
- Scheduled or automated reports
- Advanced analytics (trend, forecasting)
- Export automation (email, scheduled PDF)
- Real-time dashboards

---

## 3. Actors & Access Control

Access to reports is **read-only**.

- **Admin**
  - View all X reports within a branch
  - View Z summary (end-of-day) for a branch
  - View Z session detail for a closed session (admin-only audit view)
- **Manager**
  - View all X reports within own branch
- **Cashier**
  - View only their own X reports within own branch

---

## 4. Core Concepts

### 4.1 Final vs Provisional Data

Reports distinguish between:
- **Finalized data**: confirmed and immutable
- **Provisional data**: subject to approval or reversal

Pending voids are treated as **provisional** and are never silently excluded.

### 4.2 Branch-Scoped Policy vs Reporting Truth

Policies (VAT, FX, rounding, inventory behaviors) are **configured per branch**.

However, **reports must not recompute historical totals using “current policy”**.
- Sales reporting must aggregate **stored finalized amounts** (computed during sale finalization under the branch policy at that time).
- If UI formatting needs a branch context (e.g., showing “KHR rounding enabled”), the report UI may read the **branch policy** for display hints, but not to alter totals.

This prevents policy changes from retroactively changing financial history.

### 4.3 Frozen Branch Visibility (Read-Only History)

Branches may be **frozen** (inactive / billing-frozen / operationally disabled).  
Reporting must remain available for historical review:

- Frozen branches **must remain selectable in report filters**.
- Frozen branches must be clearly labeled (e.g., “Frozen” badge/status).
- Reports remain **read-only** and do not enable any operational actions.

---

## 5. Use Cases

### UC-1: View Cash X Reports (Branch)

**Actors:** Admin, Manager, Cashier (role-scoped)

**Preconditions:**
- User is authenticated
- Branch context is resolved (active or frozen)

**Main Flow:**
1. User opens Cash Reports → X
2. User selects branch (if applicable) and filters (date range, status)
3. System returns a list of cash sessions (X reports) in that branch:
   - session status
   - opened by name
   - opened at / closed at

**Acceptance Criteria:**
- Admin/Manager see all sessions in their branch scope.
- Cashier sees only sessions they opened.
- Report list is read-only and does not allow tampering.

---

### UC-2: View Cash X Report Detail (Session)

**Actors:** Admin, Manager, Cashier (role-scoped)

**Preconditions:**
- Session exists in the branch

**Main Flow:**
1. User opens an X report detail for a session
2. System displays:
   - status, opener, opened/closed timestamps
   - opening float
   - cash sale movement totals
   - paid-in / paid-out totals
   - expected cash (by currency)

---

### UC-3: View Cash Z Summary (End of Day)

**Actor:** Admin

**Preconditions:**
- Branch context resolved (active or frozen)
- Date selected (YYYY-MM-DD)

**Main Flow:**
1. Admin requests Z summary for a branch + date
2. System computes and returns the end-of-day totals by aggregating sessions for that day

**Acceptance Criteria:**
- Z summary is computed on request (no reactive aggregation to X changes).
- Z summary is read-only.

---

### UC-4: View Cash Z Session Detail (Closed Session)

**Actor:** Admin  

**Preconditions:**
- Cash session is closed
- Branch may be active or frozen

**Main Flow:**
1. Admin opens Z Report
2. System displays:
   - Final cash totals
   - Counted cash
   - Variance
   - Session metadata

**Acceptance Criteria:**
- Session detail is read-only
- Corrections occur via explicit future movements/adjustments (if allowed)

---

### UC-5: View Pending Void Exposure (Deferred)

**Actor:** Admin

**Preconditions:**
- Not in Capstone 1 demo scope

**Main Flow:**
- Deferred

**Acceptance Criteria:**
- Deferred

---

## 6. Reporting Rules

- VOID_PENDING sales:
  - Included in reports
  - Clearly labeled as provisional
  - Excluded from confirmed revenue
- VOIDED sales:
  - Fully excluded from all totals
- Finalized monetary totals:
  - Must be aggregated from **stored finalized values**
  - Must not be recalculated from current branch policy
- Cash reports:
  - Never retroactively changed
  - Corrections occur through future movements only (subject to cash policy)
- Frozen branches:
  - Must remain visible/selectable in reporting filters
  - Must be labeled as frozen in UI
  - Do not block viewing historical reports

---

## 7. Non-Functional Requirements

- Reports must be:
  - Deterministic
  - Auditable
  - Role-restricted
- Performance must support:
  - Daily and monthly views
  - Branch-level aggregation
- All report access must be logged via Audit Module

---

## 8. Out of Scope (Explicit)

- Cross-branch totals (tenant-wide)
- Export automation
- Real-time analytics
- Manager-level reporting
- Financial forecasting

---

## 9. Design Rationale (Summary)

This module follows mature POS principles by:
- Separating operational state from accounting truth
- Avoiding retroactive mutation of financial records
- Making provisional risk visible instead of hidden
- Preserving reporting access even when branches are frozen

This ensures trust, auditability, and long-term system integrity.

---

## Audit Events Emitted

The following events MUST be written to the Audit Log when triggered (with tenant_id, branch_id where applicable, actor_id, and relevant entity IDs):

- `REPORT_VIEWED`
- `REPORT_EXPORTED`
