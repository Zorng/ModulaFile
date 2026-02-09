# Edge Case Contract — Offline Sync

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: offline queueing + replay + idempotency + access safety under unstable connectivity
- **Primary Audience**: Frontend, Backend, QA
- **Owner(s)**: Offline Sync, Access Control, Sale/Order, Cash Session, Attendance, Policy, Audit
- **Last Updated**: 2026-02-08
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Domains**:
  - `BusinessLogic/2_domain/60_PlatformSystems/offline_sync_domain.md`
  - `BusinessLogic/2_domain/10_Identity&Authorization/accessControl_domain_consistency_patched.md`
  - `BusinessLogic/2_domain/60_PlatformSystems/policy_domain.md`
- **Related Processes**:
  - `BusinessLogic/4_process/60_PlatformSystems/60_offline_operation_queue_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/70_offline_sync_replay_process.md`

---

## Purpose

This contract locks the minimum offline behaviors needed for March:
- operations do not duplicate when replayed,
- authorization and branch status are revalidated on sync,
- the UI makes offline state visible,
- and stale cached data does not silently corrupt backend truth.

---

## Definitions / Legend

- **Offline operation**: a queued write captured while the client cannot reach the server.
- **Replay**: resubmitting queued operations when connectivity returns.
- **Stale cache**: cached menu/policy/context values that may not match server truth.

---

## Edge Case Catalog

### EC-OS-01 — Replay Must Not Create Duplicates
- **Scenario**: Client replays the same operation more than once (retries, offline replay).
- **Trigger**: Network drops or user retries.
- **Expected Behavior**:
  - Backend applies the operation exactly once using `client_op_id`.
  - Duplicate submissions return the existing result.
- **Owner**: Offline Sync + backend modules
- **March**: Yes

### EC-OS-02 — Cached Data Is Stale (Menu / Policy / Branch Status)
- **Scenario**: Cached menu/policy is outdated; branch becomes frozen while offline.
- **Trigger**: Offline sales or operations using cached data.
- **Expected Behavior**:
  - Client uses cached values to proceed, but sync revalidates on the server.
  - If server rejects due to branch frozen or invalid prerequisites, the operation is marked `FAILED` with a clear reason.
  - UI must surface the failure (no silent loss).
- **Owner**: Offline Sync + Access Control + calling module
- **March**: Yes

### EC-OS-03 — Tenant Switch with Pending Operations
- **Scenario**: User switches tenant or logs out while queue has pending operations.
- **Trigger**: Logout / tenant context change.
- **Expected Behavior**:
  - Pending operations remain bound to their original tenant/branch.
  - Sync is paused for those ops until the original tenant context is active again (or a system admin resolves).
- **Owner**: Offline Sync + Authentication
- **March**: Yes

### EC-OS-04 — Partial Sync Failure in FIFO Chain
- **Scenario**: An earlier queued operation fails (example: cash session open rejected), and later operations depend on it.
- **Trigger**: Sync replay encounters a failure in the queue.
- **Expected Behavior**:
  - Failed operation is marked `FAILED` with reason.
  - Dependent operations should not be applied silently; they must be blocked or fail with dependency error.
  - UI must surface the blocked/failure state.
- **Owner**: Offline Sync + calling modules
- **March**: Yes

### EC-OS-05 — Device Clock Skew
- **Scenario**: Offline device time is wrong.
- **Trigger**: Offline operations created with incorrect timestamps.
- **Expected Behavior**:
  - `occurred_at` is preserved for audit context but not used for ordering.
  - Backend ordering and recorded timestamps are authoritative.
- **Owner**: Offline Sync + Audit
- **March**: Yes

### EC-OS-06 — Offline Finalize Without Required Preconditions
- **Scenario**: Offline finalize occurs without an open cash session or required context.
- **Trigger**: Cashier finalizes sale offline without a valid session state.
- **Expected Behavior**:
  - Client should prevent finalize if it can detect missing prerequisites.
  - If it still occurs, backend rejects on replay with a clear reason.
- **Owner**: Sale + Cash Session + Offline Sync
- **March**: Yes

---

## Summary

For March, offline sync must be:
- idempotent under replay,
- honest about stale data,
- safe under tenant/branch context changes,
- and explicit about failures.

