# Edge Case Contract — Idempotency

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: idempotency semantics for critical writes (online + offline)
- **Primary Audience**: Backend, Frontend, QA
- **Owner(s)**: Idempotency, Sale/Order, Cash Session, Attendance, Inventory, Offline Sync, Access Control
- **Last Updated**: 2026-02-09
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Domains**:
  - `BusinessLogic/2_domain/60_PlatformSystems/idempotency_domain.md`
  - `BusinessLogic/2_domain/60_PlatformSystems/offline_sync_domain.md`
  - `BusinessLogic/2_domain/10_Identity&Authorization/accessControl_domain_consistency_patched.md`
- **Related Processes**:
  - `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`

---

## Purpose

This contract ensures retries do not cause duplicate business effects, whether offline or online.

---

## Definitions / Legend

- **Idempotency key**: token representing one user intent.
- **Duplicate**: same key + same payload.
- **Conflict**: same key + different payload.

---

## Edge Case Catalog

### EC-IDEM-01 — Duplicate Submission (Online Retry)
- **Scenario**: A finalize request is retried after a timeout.
- **Trigger**: User taps twice or client retries.
- **Expected Behavior**:
  - The second request returns the already-applied result.
  - No duplicate state change occurs.
- **Owner**: Idempotency + calling module
- **March**: Yes

### EC-IDEM-02 — Offline Replay Duplicates
- **Scenario**: Offline queue replays the same operation multiple times.
- **Trigger**: Reconnect + retry loops.
- **Expected Behavior**:
  - Backend applies the operation exactly once.
  - Duplicates return the existing result.
- **Owner**: Offline Sync + Idempotency
- **March**: Yes

### EC-IDEM-03 — Same Key, Different Payload (Conflict)
- **Scenario**: A bug reuses an idempotency key with different intent.
- **Trigger**: Same key but different payload hash.
- **Expected Behavior**:
  - Reject as conflict with a clear error.
  - Do not apply any new state change.
- **Owner**: Idempotency
- **March**: Yes

### EC-IDEM-04 — Partial Failure (State Changed but Idempotency Record Missing)
- **Scenario**: A transaction crashes after applying state but before recording the idempotency result.
- **Trigger**: DB failure mid-transaction.
- **Expected Behavior**:
  - State change and idempotency record must be atomic.
  - If record fails, state change must not persist.
- **Owner**: Calling module + Idempotency
- **March**: Yes

### EC-IDEM-05 — Authorization Changes Between Retries
- **Scenario**: A user retries an operation after permissions are revoked.
- **Trigger**: Retry after role/assignment change.
- **Expected Behavior**:
  - If the original request already applied, return the stored result.
  - If it did not apply, re-run Access Control and deny if no longer authorized.
- **Owner**: Access Control + Idempotency
- **March**: Yes

---

## Summary

For March, idempotency must prevent duplicates across **both online retries and offline replay**, with conflicts detected and state changes applied exactly once.

