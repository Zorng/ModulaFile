# Edge Case Contract — Audit Logging

## Metadata
- **Contract Type**: Edge Cases
- **Scope**: audit event recording + idempotent ingestion + access-safe viewing across tenant/branch contexts
- **Primary Audience**: Frontend, Backend, QA
- **Owner(s)**: Audit Logging, Access Control, Offline Sync, Sale/Order, Cash Session, Inventory, HR, Tenant/Branch
- **Last Updated**: 2026-02-08
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred
- **Related Domains**:
  - `BusinessLogic/2_domain/60_PlatformSystems/audit_domain.md`
  - `BusinessLogic/2_domain/10_Identity&Authorization/accessControl_domain_consistency_patched.md`
  - `BusinessLogic/2_domain/20_OrgAccount/branch_domain.md`
- **Related ModSpecs**:
  - `BusinessLogic/5_modSpec/60_PlatformSystems/audit_module.md`
  - `BusinessLogic/5_modSpec/60_PlatformSystems/offlineSync_module.md`

---

## Purpose

This contract locks the minimum “audit must be trustworthy” behaviors for March:
- state changes are not allowed to happen silently without evidence,
- duplicates under retry/offline replay do not inflate logs,
- access is safe (no cross-tenant leakage),
- and time semantics remain explainable under clock skew.

---

## Definitions / Legend

- **Occurred time**: when the action happened (client/server perspective).
- **Recorded time**: when the backend persisted the audit record.
- **Idempotency**: repeated submits do not create duplicates.

---

## Edge Case Catalog

### EC-AUD-01 — Audit Write Failure Must Not Create “Unaudited State”
- **Scenario**: A state-changing operation succeeds but audit insert fails (or vice versa).
- **Trigger**: DB constraint issues, unexpected schema changes, partial failures.
- **Expected Behavior**:
  - For state-changing operations, persist the business state change and the audit record **atomically** (same transaction).
  - If the audit write fails, the state change must be rolled back and the operation fails.
  - Observational audit events (example: report viewed) may be best-effort and must not block the read surface.
- **Owner**: Business module + Audit module
- **March**: Yes

### EC-AUD-02 — Offline Replay / Retry Must Not Duplicate Audit Events
- **Scenario**: Client retries the same operation due to network drops or offline sync replay.
- **Trigger**: Operation is submitted more than once.
- **Expected Behavior**:
  - Audit ingestion must be idempotent using a stable operation ID / idempotency key.
  - Duplicate ingestion returns success (or no-op) without creating a second audit entry.
- **Owner**: Offline Sync + Audit
- **March**: Yes

### EC-AUD-03 — Client Clock Skew (Occurred Time Cannot Be Fully Trusted)
- **Scenario**: Device time is wrong; offline events have incorrect timestamps.
- **Trigger**: Offline operations recorded with client time.
- **Expected Behavior**:
  - Store both `occurred_at` and `recorded_at`.
  - Default ordering in audit viewing should use `recorded_at` (server time) while still displaying `occurred_at` if present.
  - Never “rewrite” occurred times on sync; preserve original value, but be clear it may be unreliable.
- **Owner**: Audit + Offline Sync + Frontend
- **March**: Yes

### EC-AUD-04 — Actor Leaves / Is Disabled (History Must Stay Traceable)
- **Scenario**: A membership is revoked after actions were performed.
- **Trigger**: Viewing old audit entries.
- **Expected Behavior**:
  - Historical audit entries remain visible and immutable.
  - Audit entries must still reference the actor identity (at minimum `actor_id`), even if the user can no longer log in.
  - UI may show “revoked member” (or equivalent) but must preserve traceability.
- **Owner**: Identity/HR + Audit
- **March**: Yes

### EC-AUD-05 — Branch Frozen / Tenant Frozen Rejections Must Be Auditable
- **Scenario**: Actions are blocked because the branch is frozen or tenant is frozen.
- **Trigger**: Any state-changing attempt on frozen context.
- **Expected Behavior**:
  - Operation is denied (fail closed) and an audit entry is recorded with outcome `REJECTED`.
  - Reason code should be the Access Control `reason_code` when denial is due to authorization checks.
- **Owner**: Access Control + Audit + calling module
- **March**: Yes

### EC-AUD-06 — Audit Log Viewing Must Not Leak Cross-Tenant Data
- **Scenario**: A user attempts to query audit logs without the right tenant context or permissions.
- **Trigger**: Audit log view/query request.
- **Expected Behavior**:
  - Deny when tenant context is missing.
  - Deny when actor lacks `audit.view` for the tenant.
  - Never return records from other tenants.
- **Owner**: Access Control + Audit
- **March**: Yes

### EC-AUD-07 — Sensitive Data Must Not Be Stored in Audit Metadata
- **Scenario**: A module attempts to write sensitive content (passwords, OTPs, raw secrets).
- **Trigger**: Audit event creation.
- **Expected Behavior**:
  - Reject or sanitize fields that contain credentials/secrets.
  - Prefer entity IDs and stable keys over raw payload dumps.
- **Owner**: All modules + Audit
- **March**: Yes (at minimum: do not log secrets)

### EC-AUD-08 — Large Volumes (Pagination and Filters Are Required)
- **Scenario**: A tenant generates many audit events (busy branch).
- **Trigger**: Viewing audit logs over long time windows.
- **Expected Behavior**:
  - Query must be paginated.
  - Filters by date range and branch are supported.
  - “Export everything” is out of scope for March.
- **Owner**: Audit + Frontend
- **March**: Yes (pagination + filters)

---

## Summary

For March, Audit Logging must be:
- atomic with state-changing operations,
- idempotent under offline replay,
- access-safe and tenant-isolated,
- and explicit about time semantics.
