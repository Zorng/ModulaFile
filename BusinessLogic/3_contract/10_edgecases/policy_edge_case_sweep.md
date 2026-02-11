# Edge Case Contract — Policy (Branch Configuration)

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: Branch-scoped policy (tax/currency + minimal sale workflow toggle); update/read behavior; interactions with Sale snapshot + Reporting
- **Primary Audience**: Frontend, Backend, QA
- **Owner(s)**: Policy, Sale/Order, Reporting, Access Control, Audit, Tenant/Branch, Offline Sync
- **Last Updated**: 2026-02-08
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Domains**:
  - `BusinessLogic/2_domain/60_PlatformSystems/policy_domain.md`
  - `BusinessLogic/2_domain/10_Identity&Authorization/accessControl_domain_consistency_patched.md`
  - `BusinessLogic/2_domain/20_OrgAccount/branch_domain.md`
  - `BusinessLogic/2_domain/50_Reporting/reporting_domain.md`
- **Related ModSpecs**:
  - `BusinessLogic/5_modSpec/60_PlatformSystems/policy_module.md`
  - `BusinessLogic/5_modSpec/40_POSOperation/sale_module_patched.md`
  - `BusinessLogic/5_modSpec/50_Reporting/report_module.md`
  - `BusinessLogic/5_modSpec/60_PlatformSystems/offlineSync_module.md`
  - `BusinessLogic/5_modSpec/60_PlatformSystems/audit_module.md`

---

## Purpose

This contract locks the minimum edge-case behavior needed for Policy to be trustworthy:
- branch scope is explicit,
- invalid configurations are rejected safely,
- updates do not rewrite history,
- and offline/stale states are handled honestly.

It prevents policy drift where different modules apply VAT/FX/rounding inconsistently.

---

## Definitions / Legend

- **Policy**: branch-scoped configuration used by Sale:
  - tax/currency (VAT, FX, rounding)
  - sale workflow toggle(s) such as pay-later enablement
- **Finalized sale snapshot**: the immutable sale totals and inputs captured at finalize time.
- **Stale**: the client is displaying cached policy values that may not match the server’s latest values.

---

## Edge Case Catalog

### EC-POL-01 — Policy Changes Must Not Rewrite Past Sales/Receipts/Reports
- **Scenario**: Admin updates VAT/FX/rounding after sales already happened.
- **Trigger**: Viewing past receipts or historical reports; syncing offline-finalized sales.
- **Expected Behavior**:
  - Past receipts and past report totals remain based on the finalized sale snapshot.
  - Reporting must aggregate from stored finalized values; no retroactive recompute using current policy.
- **Owner**: Sale/Order (snapshots) + Receipt (render from snapshots) + Reporting
- **March**: Yes

### EC-POL-02 — Branch Context Is Mandatory (Scope Must Be Explicit)
- **Scenario**: Client attempts to read or update policy without a `branch_id`, or with a branch the actor cannot access.
- **Trigger**: Policy view/update request.
- **Expected Behavior**:
  - Deny when `branch_id` is missing for policy operations.
  - Deny when the actor lacks access to the branch context.
- **Owner**: Access Control + Policy
- **March**: Yes

### EC-POL-03 — Invalid Values Must Be Rejected (Fail Closed)
- **Scenario**: Admin submits values outside allowed ranges.
- **Trigger**: Policy update.
- **Expected Behavior**:
  - Reject update with a clear reason (validation error).
  - No partial writes (atomicity preserved).
  - No downstream module sees half-updated configurations.
- **Owner**: Policy (validation + atomicity) + QA (coverage)
- **March**: Yes

### EC-POL-04 — Concurrent Updates Must Not Corrupt Policy
- **Scenario**: Two admins update policy around the same time (same branch).
- **Trigger**: Near-simultaneous updates.
- **Expected Behavior**:
  - Backend must apply each update atomically.
  - Audit log must capture each update (old → new).
  - UI should refresh from server after update to avoid stale display.
  - If conflicting updates occur on the same field, last-write-wins is acceptable for March, but must be auditable.
- **Owner**: Policy + Audit + Frontend (refresh)
- **March**: Yes (last-write-wins acceptable)
- **Later**: optimistic concurrency (ETag/version) to reduce surprises

### EC-POL-05 — Update During Live Operations (Mid-Shift Changes)
- **Scenario**: Admin updates VAT/FX/rounding while cashiers are actively selling.
- **Trigger**: Policy update during an OPEN cash session.
- **Expected Behavior**:
  - New finalized sales use the latest policy at finalize time.
  - Already-finalized sales remain unchanged.
  - If a cart is open while policy changes, the system must avoid “silent mismatch”:
    - either refresh totals before finalize, or
    - server recomputes from current policy on finalize (preferred for correctness).
- **Owner**: Sale/Order (finalize correctness) + Policy (source values) + Frontend (refresh behavior)
- **March**: Yes (must not corrupt totals)

### EC-POL-06 — Frozen Branch (Read vs Write)
- **Scenario**: Branch is frozen/suspended.
- **Trigger**: Viewing or updating policy for a frozen branch.
- **Expected Behavior**:
  - Policy remains readable (administration + audit).
  - Policy updates are denied if branch is frozen (unless an explicit “admin override” process exists later).
- **Owner**: Branch domain + Access Control + Policy
- **March**: Yes (read ok, write denied)

### EC-POL-07 — Offline / Stale Policy Values Must Be Explicit
- **Scenario**: Device is offline or recently reconnected; cached policy may be stale.
- **Trigger**: Viewing policy or finalizing sales offline.
- **Expected Behavior**:
  - For March, policy updates should be blocked while offline (do not queue policy edits silently).
  - Policy reads may use cached values, but staleness must be explicit.
  - Offline-finalized sales must snapshot the policy inputs used; on sync, history must not be rewritten by current policy.
- **Owner**: Policy + Offline Sync + Sale/Order
- **March**: Yes (explicit degradation)

### EC-POL-08 — Missing Policy Record Falls Back to Defaults (With Signal)
- **Scenario**: A branch exists but policy record is missing due to migration/bug.
- **Trigger**: Policy resolution for sale/reporting.
- **Expected Behavior**:
  - Resolve using default values (fail-safe) to avoid blocking operations unexpectedly.
  - Emit an audit/system signal so the missing-record issue is discoverable and fixable.
- **Owner**: Policy + Audit/Monitoring
- **March**: Yes (defaults + visibility)

### EC-POL-09 — Toggling Pay-Later Must Not Strand Open Tickets
- **Scenario**: Admin disables pay-later while there are unpaid tickets currently open in the branch.
- **Trigger**: Policy update `saleAllowPayLater: true → false`.
- **Expected Behavior (March baseline)**:
  - New pay-later entry points are denied (no new open ticket creation; no add-items batches).
  - Existing open tickets remain settleable (checkout/close) so operations can finish safely.
  - The system must not delete or mutate open tickets due to the policy toggle.
- **Owner**: Policy (toggle truth) + POSOperation processes (enforcement + safe closure)
- **March**: Yes

---

## Summary

For March, policy must be:
- explicitly branch-scoped,
- validated and atomic on update,
- historically safe (no retroactive recompute),
- and honest about offline/stale states.

_End of Policy edge case contract_
