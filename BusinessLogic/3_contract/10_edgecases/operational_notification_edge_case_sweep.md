# Edge Case Contract — Operational Notification (In-App Signals)

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: In-app operational notifications; dedupe/replay safety; stale-state safety; access control enforcement
- **Primary Audience**: Frontend, Backend, QA
- **Owner(s)**: Operational Notification, Access Control, Offline Sync, Audit, POSOperation (Void, CashSession)
- **Last Updated**: 2026-02-08
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Stories**:
  - `BusinessLogic/1_stories/correcting_mistakes/responding_to_operational_notifications.md`
  - `BusinessLogic/1_stories/correcting_mistakes/voiding_a_sale.md`
  - `BusinessLogic/1_stories/handling_money/closing_cash_drawer.md`
- **Related Domains**:
  - `BusinessLogic/2_domain/60_PlatformSystems/operational_notification_domain.md`
  - `BusinessLogic/2_domain/10_Identity&Authorization/accessControl_domain_consistency_patched.md`
- **Related Processes**:
  - `BusinessLogic/4_process/30_POSOperation/20_void_sale_orch.md`
  - `BusinessLogic/4_process/60_PlatformSystems/30_operational_notification_emission_process.md`
- **Related ModSpecs**:
  - `BusinessLogic/5_modSpec/60_PlatformSystems/operationalNotification_module.md`
  - `BusinessLogic/5_modSpec/40_POSOperation/sale_module_patched.md`
  - `BusinessLogic/5_modSpec/40_POSOperation/cashSession_module_patched_v2.md`

---

## Purpose

This contract locks the minimum edge-case behavior required for operational notifications to be safe:
- notifications are **signals**, not workflow authority
- duplicates do not occur under retries/offline replay
- access boundaries are enforced
- stale notifications do not cause invalid actions

---

## Definitions / Legend

- **Notification**: an in-app inbox item referencing a subject (sale, cash session, etc.)
- **Recipient**: a user who is eligible to see a notification
- **Dedupe key**: deterministic idempotency anchor for notification creation
- **Stale notification**: notification exists but the underlying business state has changed since it was created

---

## Edge Case Catalog

### EC-ON-01 — Duplicate Notifications Under Retry / Offline Replay
- **Scenario**: The same trigger event is delivered multiple times (network retry, job retry, offline sync replay).
- **Trigger**: Notification emission runs more than once for the same business transition.
- **Expected Behavior**:
  - Notification creation is idempotent using a deterministic `dedupe_key`.
  - Duplicate trigger attempts return success without creating a second notification.
- **Owner**: Operational Notification + Offline Sync
- **March**: Yes

### EC-ON-02 — Notification Creation Failure Must Not Break Business Truth
- **Scenario**: Notification persistence fails (DB issue, transient error) during an operational workflow.
- **Trigger**: Emission step errors.
- **Expected Behavior**:
  - The underlying business transition remains correct and committed.
  - The system may retry notification emission later.
  - UIs must still expose the underlying state via normal lists (example: pending void requests list), not only via notifications.
- **Owner**: Emitting process owner (Void/CashSession/etc.) + Operational Notification
- **March**: Yes (best-effort)

### EC-ON-03 — Stale Notifications Must Not Authorize Invalid Actions
- **Scenario**: A user taps a notification and attempts an action, but state has already changed (another manager approved/rejected).
- **Trigger**: Action initiated from a notification deep link.
- **Expected Behavior**:
  - UI/API re-check current state on open and on action.
  - If already resolved, action becomes a no-op with a clear message (“Already resolved”).
- **Owner**: Source workflow (Void/CashSession) + Access Control
- **March**: Yes

### EC-ON-04 — Multiple Approvers Race (Only One Can Succeed)
- **Scenario**: Two managers/admins try to approve the same void request.
- **Trigger**: Concurrent approval attempts.
- **Expected Behavior**:
  - Only one approval is accepted; others receive a deterministic conflict/no-op response.
  - Notification remains a signal; it does not guarantee the actor “owns” the request.
- **Owner**: Void orchestration + Operational Notification (non-authoritative)
- **March**: Yes

### EC-ON-05 — Access Changes After Notification Creation
- **Scenario**: Recipient loses tenant/branch access after a notification was created (role change, branch assignment revoked).
- **Trigger**: User opens inbox after access change.
- **Expected Behavior**:
  - The user must not be able to view or act on notifications they are no longer authorized for.
  - UI may hide such notifications or show an “access removed” placeholder (implementation choice), but must not leak data.
- **Owner**: Access Control + Operational Notification
- **March**: Yes

### EC-ON-06 — Cross-Tenant / Cross-Branch Leakage Is Forbidden
- **Scenario**: Notification recipients accidentally include users from another tenant/branch.
- **Trigger**: Recipient computation bug.
- **Expected Behavior**:
  - Hard isolation: recipients must be computed strictly within `(tenant_id, branch_id)` context.
  - Inbox queries must also enforce tenant/branch boundaries.
- **Owner**: Operational Notification + Access Control + QA
- **March**: Yes

### EC-ON-07 — Subject Missing or Deleted
- **Scenario**: Notification references a subject that no longer exists or is not accessible (data repair, archival, permission mismatch).
- **Trigger**: User opens the notification deep link.
- **Expected Behavior**:
  - UI shows a safe error state (“Item no longer available”) without exposing private information.
  - Notification remains in inbox (auditable history), but cannot be acted on.
- **Owner**: Operational Notification + Source module
- **March**: Yes (safe degradation)

### EC-ON-08 — Frozen Branch: Notifications Visible, Actions Denied
- **Scenario**: Branch becomes frozen after notifications were created.
- **Trigger**: Recipient opens notification and tries to act.
- **Expected Behavior**:
  - Notifications remain visible (read-only awareness).
  - Operational actions remain blocked by branch freeze rules.
- **Owner**: Branch domain + Access Control + Source workflow
- **March**: Yes

### EC-ON-09 — Offline / Stale Inbox Must Be Explicit
- **Scenario**: Device is offline or recently reconnected; inbox state may be stale.
- **Trigger**: User opens notification inbox while offline.
- **Expected Behavior**:
  - Inbox may show cached items, but staleness must be explicit.
  - Users must not be misled into thinking they are seeing all current notifications.
- **Owner**: Offline Sync + Operational Notification UX
- **March**: Yes (explicit degradation)

### EC-ON-10 — Variance Is Reported Without Thresholding (March)
- **Scenario**: Cash session is closed and there is variance (small or large).
- **Trigger**: CashSession transitions to `CLOSED`.
- **Expected Behavior**:
  - For March, do not introduce tenant-specific variance thresholds.
  - Include variance values in the cash-session-closed notification payload/body so humans can evaluate significance.
- **Owner**: Cash Session + Operational Notification
- **March**: Yes

---

## Summary

Operational notifications must remain:
- idempotent under replay,
- best-effort (never a correctness dependency),
- access-safe,
- and stale-state safe (state is authority).

_End of Operational Notification edge case contract_
