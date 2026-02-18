# Edge Case Contract — Subscription, Billing, and Entitlements

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: Subscription enforcement + entitlements gating across modules (branch-scoped pricing; tenant billing state)
- **Primary Audience**: Backend, Frontend, QA
- **Owner(s)**: PlatformSystems (Subscription & Entitlements), OrgAccount (Tenant/Branch), Access Control
- **Last Updated**: 2026-02-18
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Docs**:
  - `BusinessLogic/2_domain/60_PlatformSystems/subscription_entitlements_domain.md`
  - `BusinessLogic/2_domain/20_OrgAccount/branch_domain.md`
  - `BusinessLogic/2_domain/20_OrgAccount/tenant_domain_consistency_patched.md`
  - `BusinessLogic/2_domain/10_Identity&Authorization/accessControl_domain_consistency_patched.md`

---

## Purpose

Lock the “billing guard rails” edge cases so operational modules do not implement ad-hoc enforcement.

This contract focuses on three things:
- subscription state (`ACTIVE` / `PAST_DUE` / `FROZEN`)
- entitlement enforcement (`ENABLED` / `READ_ONLY` / `DISABLED_VISIBLE`)
- upgrade/downgrade timing and safety boundaries (`DOWNGRADE_PENDING`)

---

## Definitions / Legend

- **Operational write**: any action that changes business truth (sale finalize, inventory adjust, start work, open cash session, etc.).
- **Read-only**: viewing history/records is allowed; all writes are blocked.
- **Branch activation**: first paid branch provisioning flow when tenant has zero branches.
- **Seat**: concurrent operator seat for a branch, consumed at `START_WORK`.

---

## Edge Case Catalog

### EC-SUB-01 — Zero-Branch Tenant Attempts to Operate POS
- **Scenario**: A first-time tenant signs in but has no branches yet.
- **Trigger**: User tries to enter any operational module (sale, inventory, workforce) without an active branch.
- **Expected Behavior**:
  - System must not present branch-scoped operations as “available”.
  - User is guided to **first-branch activation** (draft → pay → provision).
- **Owner**: OrgAccount + Subscription/Entitlements
- **March**: Yes

---

### EC-SUB-02 — Payment Completed but Branch Not Yet Provisioned (Activation Race)
- **Scenario**: Payment succeeds, but branch provisioning is delayed (network/job delay).
- **Trigger**: Payment verification returns success, but the UI still shows “no branch”.
- **Expected Behavior**:
  - Show “Activating…” state and retry provisioning status.
  - Do not allow duplicate payments for the same activation invoice.
- **Owner**: Subscription/Entitlements + Idempotency (platform)
- **March**: Yes

---

### EC-SUB-03 — `PAST_DUE` Grace Window (24h) Behavior
- **Scenario**: Invoice unpaid at renewal.
- **Trigger**: Tenant enters `PAST_DUE`.
- **Expected Behavior**:
  - Operations continue (do not brick the counter).
  - UI must show prominent warnings and a clear “Pay now” action.
  - No “silent degraded” behavior; warnings must be explicit.
- **Owner**: Subscription/Entitlements + UX (frontend)
- **March**: Yes

---

### EC-SUB-04 — `FROZEN` State Blocks Operational Writes (Read-Only Still Allowed)
- **Scenario**: Tenant remains unpaid beyond grace window.
- **Trigger**: Tenant enters `FROZEN`.
- **Expected Behavior**:
  - All operational writes are blocked (sale finalize, inventory writes, staff start work, cash session open/close, etc.).
  - Read-only access is allowed where safe (history, reports, invoices).
  - Error messaging must be consistent and explain “subscription frozen”.
- **Owner**: Subscription/Entitlements + Access Control
- **March**: Yes

---

### EC-SUB-05 — Seat Limit Reached at `START_WORK`
- **Scenario**: Workforce is enabled, but all operator seats are already in use.
- **Trigger**: Staff/admin attempts `START_WORK`.
- **Expected Behavior**:
  - Deny starting work with a clear “seat limit reached” message.
  - Provide operational guidance (who is active) and upgrade path (increase seats).
- **Owner**: Workforce process + Subscription/Entitlements
- **March**: Yes

---

### EC-SUB-06 — Disable Workforce While Staff Are Active (`DOWNGRADE_PENDING`)
- **Scenario**: Owner/admin requests Workforce downgrade while staff are working.
- **Trigger**: Branch enters `DOWNGRADE_PENDING`.
- **Expected Behavior**:
  - Block new staff check-ins and new staff cash sessions.
  - Allow existing sessions to end/close cleanly.
  - Workforce becomes OFF only at safe boundary (no active staff attendance; no open staff cash sessions).
- **Owner**: Workforce + Subscription/Entitlements
- **March**: Yes

---

### EC-SUB-07 — Workforce OFF: Staff Can Still Authenticate but Cannot Operate That Branch
- **Scenario**: Workforce is OFF for a branch that previously had staff.
- **Trigger**: A staff member logs in and selects tenant/branch.
- **Expected Behavior**:
  - Staff should not be able to select/operate that branch as a separate actor.
  - History remains viewable by owner/admin (read-only).
- **Owner**: Subscription/Entitlements + Access Control + Authentication (branch context resolution)
- **March**: Yes

---

### EC-SUB-08 — Disable Inventory: Selling Continues (No Deduction; Inventory Writes Blocked)
- **Scenario**: Inventory is disabled for a branch that has historical inventory data.
- **Trigger**: Inventory entitlement becomes read-only/disabled.
- **Expected Behavior**:
  - Selling continues; items behave as NOT_TRACKED (no deductions).
  - Inventory writes are blocked; history remains read-only.
  - Composition UI is visible but disabled with upgrade prompt.
- **Owner**: Inventory + Menu + Subscription/Entitlements
- **March**: Yes

---

### EC-SUB-09 — Upgrade Mid-Cycle: Charge Now vs Unlock Now
- **Scenario**: Owner enables a module mid-cycle.
- **Trigger**: “Enable Inventory/Workforce” action.
- **Expected Behavior**:
  - Invoice/charge is issued immediately (proration).
  - Capability unlock is immediate after payment confirmation (KHQR auto-verify in March).
  - If payment fails/cancels, capability remains locked.
- **Owner**: Subscription/Entitlements
- **March**: Yes

---

### EC-SUB-10 — Entitlement Drift: UI Shows Enabled but Backend Denies
- **Scenario**: Client cache says module enabled but backend snapshot says locked (stale cache).
- **Trigger**: A write is attempted while entitlement is no longer enabled.
- **Expected Behavior**:
  - Backend denial is authoritative.
  - Client refreshes entitlement snapshot and updates UI state.
- **Owner**: Subscription/Entitlements + Offline Sync (cache) + Frontend
- **March**: Yes

---

### EC-SUB-11 — Branch Archive/Delete Does Not Restore Reusable Branch Entitlement
- **Scenario**: Owner/admin archives or deletes an existing branch after it was activated through paid subscription flow.
- **Trigger**: User attempts to activate another branch expecting free reuse of archived/deleted branch activation entitlement.
- **Expected Behavior**:
  - Archiving/deleting a branch does not mint reusable free branch entitlement by default.
  - A new branch still requires `additional branch subscription` activation flow and payment confirmation.
  - Reusable behavior is allowed only if explicitly defined by billing policy (outside March baseline).
- **Owner**: Subscription/Entitlements + OrgAccount
- **March**: Yes

---

## Summary

For March, subscription enforcement must be predictable:
- warn during `PAST_DUE`,
- freeze cleanly after grace (`FROZEN`),
- enforce seats at `START_WORK`,
- never allow “unpaid branch” operations,
- and treat each additional branch as a paid activation (no implicit slot reuse).
