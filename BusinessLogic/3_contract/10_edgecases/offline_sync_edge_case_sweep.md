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
  - `BusinessLogic/4_process/60_PlatformSystems/65_offline_sync_pull_hydration_process.md`
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

### EC-OS-01B — Client Operation ID Reused with Different Payload
- **Scenario**: A client (bug or malicious) reuses the same `client_op_id` for a different payload.
- **Trigger**: Replay/push receives `client_op_id` that already exists but the payload fingerprint differs.
- **Expected Behavior**:
  - Backend rejects deterministically (no partial apply).
  - Return a specific reason (example: `CLIENT_OP_ID_REUSE_WITH_DIFFERENT_PAYLOAD`) so the client can surface a non-retryable error.
- **Owner**: Offline Sync + Idempotency gate
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

### EC-OS-07 — Pull Cursor Invalid (Reset Required)
- **Scenario**: Client stores a cursor but the server can no longer honor it (feed retention window, reset, or token corruption).
- **Trigger**: Client calls pull sync with an invalid cursor.
- **Expected Behavior**:
  - Server rejects deterministically with a “cursor invalid / reset required” reason.
  - Client performs a branch rehydration reset (bootstrap pull with no cursor) and then resumes incremental pulls.
  - No silent partial hydration with missing ranges.
- **Owner**: Offline Sync + backend feed producer
- **March**: Yes

### EC-OS-08 — Module Scope Drift (Scope Hash Mismatch)
- **Scenario**: Client changes requested `module_scopes` (or the app version changes scope composition) but attempts to reuse the old cursor.
- **Trigger**: Pull sync request where `cursor.module_scope_hash` != `hash(module_scopes)`.
- **Expected Behavior**:
  - Server rejects with “scope mismatch / reset required”.
  - Client rehydrates using the new scope set (bootstrap) and records the new scope hash.
- **Owner**: Offline Sync
- **March**: Yes

### EC-OS-09 — Tombstones Must Be Applied
- **Scenario**: Server emits a TOMBSTONE (hard removal) but the client ignores it or cannot apply it.
- **Trigger**: Pull sync returns `op_kind = TOMBSTONE`.
- **Expected Behavior**:
  - Client must remove the local record (or mark deleted) so the entity does not reappear in active pickers.
  - Applying a tombstone for a missing local entity is a no-op (must not error).
- **Owner**: Offline Sync + affected module
- **March**: Yes

### EC-OS-10 — Fan-Out Correctness (Tenant-Scoped Catalog → Branch Streams)
- **Scenario**: A tenant-scoped catalog change (e.g., menu catalog update) must be visible for all active branches, but a device only syncs one branch stream.
- **Trigger**: Admin updates tenant-scoped catalog data.
- **Expected Behavior**:
  - The change is fanned out into each active branch stream.
  - A device syncing Branch A sees the update when pulling Branch A; it does not need to poll tenant scope separately.
- **Owner**: Offline Sync + catalog producers (Menu/Policy/etc.)
- **March**: Yes

### EC-OS-11 — Nudge Channel Lost (Convergence Still Required)
- **Scenario**: The device does not receive sync nudges (SSE/WebSocket), or the stream disconnects.
- **Trigger**: Connectivity issues or background suspension.
- **Expected Behavior**:
  - Client still converges via pull-on-resume/reconnect and periodic pull while online.
  - No correctness dependence on continuous streaming.
- **Owner**: Offline Sync + client UX
- **March**: Yes

---

## Summary

For March, offline sync must be:
- idempotent under replay,
- honest about stale data,
- safe under tenant/branch context changes,
- and explicit about failures.
