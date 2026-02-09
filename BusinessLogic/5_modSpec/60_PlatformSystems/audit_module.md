# Audit Logging Module — Core Module (Capstone 1)

**Version:** 1.3  
**Status:** Patched (event catalog anti-drift; Access Control scope clarified)  
**Module Type:** Core Module  
**Depends on:** Access Control (Core), Tenant & Branch Context (Core), Offline Sync (replay safety)  
**Used by:** All state-changing modules (Sale, CashSession, Inventory, Menu, Discount, HR, Policy, Reporting, OrgAccount)

---

## 1. Purpose

The Audit Logging module provides a **tamper-resistant, immutable record of business-critical actions** performed within Modula.

It exists to answer, reliably:

> Who did what, when, in which tenant/branch, and with what outcome?

Audit logs are **not analytics** and not debugging logs.

Authoritative references:
- Story: `BusinessLogic/1_stories/saas_governance/reviewing_audit_logs_for_accountability.md`
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/audit_domain.md`
- Edge cases: `BusinessLogic/3_contract/10_edgecases/audit_edge_case_sweep.md`
- Processes:
  - `BusinessLogic/4_process/60_PlatformSystems/40_audit_event_recording_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/50_audit_log_query_process.md`

---

## 2. Scope & Boundaries

### Includes
- Recording **successful state changes** across modules.
- Recording **meaningful rejections/failures** (authorization denial, branch frozen, validation failures, missing prerequisites).
- Idempotent ingestion under retries and offline sync replay.
- Branch-scoped and tenant-scoped events.
- Privileged read access for governance users (Owner/Admin).

### Excludes (Capstone 1)
- KPI dashboards and derived analytics (Reporting owns aggregates).
- Real-time alerts or coordination (Operational Notification owns in-app signals).
- Editable/deletable logs.
- External exports (SIEM, CSV exports, integrations).

---

## 3. Visibility & Access Control (March Baseline)

- Only **Owner/Admin** may view audit logs.
- Managers/Cashiers have no audit log access in Capstone 1.

Access is enforced via Access Control using the action key:
- `audit.view` (TENANT-scoped for March governance visibility)

Audit logs are read-only and cannot be modified, deleted, or backdated.

---

## 4. Core Concepts

### 4.1 AuditEvent (Entity)

An audit event represents a single business action attempt with an outcome.

Minimum fields (conceptual):
- `event_id` (unique)
- `event_type` (stable string; defined by the calling module ModSpec)
- `tenant_id` (required for in-tenant business actions)
- `branch_id` (required when the underlying action is branch-scoped)
- `actor_id` (or `SYSTEM`)
- `occurred_at` (attempt time)
- `recorded_at` (persistence time)
- `outcome` (`SUCCESS | REJECTED | FAILED`)
- `reason_code` (required when outcome is REJECTED/FAILED when possible)
- `entity_refs` (IDs only; no sensitive payload dumps)
- `metadata` (small, non-sensitive context)
- `idempotency_key` (dedupe anchor for replay safety)

### 4.2 Outcomes
- `SUCCESS`: state changed successfully, or an observational event was recorded.
- `REJECTED`: denied before state mutation (authorization, validation, branch frozen, missing prerequisite).
- `FAILED`: attempted but failed due to system failure.

### 4.3 Reason Codes (Alignment Rule)

When a rejection is caused by Access Control, record the Access Control `reason_code` as the audit `reason_code`.

Baseline non-authorization reason codes:
- `VALIDATION_FAILED`
- `DEPENDENCY_MISSING`
- `BUSINESS_RULE_BLOCKED`

---

## 5. Event Catalog (Anti-Drift Rule)

Audit Logging does not keep a single authoritative “master list” of every event type.

Source of truth:
- Each module’s ModSpec contains the list of audit events it emits.

This prevents central catalog drift as modules evolve.

### Capstone 1 Event Index (Pointers)

- Sale: `BusinessLogic/5_modSpec/40_POSOperation/sale_module_patched.md`
- Cash Session: `BusinessLogic/5_modSpec/40_POSOperation/cashSession_module_patched_v2.md`
- Inventory: `BusinessLogic/5_modSpec/40_POSOperation/inventory_module_patched.md`
- Menu: `BusinessLogic/5_modSpec/40_POSOperation/menu_module_patched.md`
- Discount: `BusinessLogic/5_modSpec/40_POSOperation/discount_module_patched.md`
- Attendance: `BusinessLogic/5_modSpec/30_HR/attendance_module.md`
- Policy: `BusinessLogic/5_modSpec/60_PlatformSystems/policy_module.md`
- Reporting (observational): `BusinessLogic/5_modSpec/50_Reporting/report_module.md`

OrgAccount and IdentityAccess processes also require auditability for governance changes (membership changes, staff provisioning).
Their event names are defined by their process/modspec artifacts and must remain stable once introduced.

---

## 6. Requirements

- R1: Every successful state-changing business action must emit an audit event.
- R2: Meaningful rejected/failed attempts must also be auditable.
- R3: Audit events are immutable once stored (append-only).
- R4: State-changing operations must persist audit records atomically with the state change.
- R5: Offline/retry replay must not duplicate audit entries (idempotency key required).
- R6: Audit logs must not contain secrets (credentials, OTPs, raw confidential payloads).

---

## 7. Acceptance Criteria

- AC-1: Owner/Admin can view audit logs for their tenant.
- AC-2: Managers/Cashiers cannot access audit logs.
- AC-3: Successful state changes appear in audit logs with actor, context, and outcome.
- AC-4: Rejected actions appear with outcome + reason code when possible.
- AC-5: Offline/retried operations do not create duplicates.
- AC-6: Audit entries cannot be edited or deleted.

---

## 8. Retention & Compliance

- Audit logs are retained for the duration of the tenant’s subscription (Capstone 1).
- Retention configuration is out of scope for March.
- Audit data is isolated per tenant and never shared across tenants.

---

## 9. Out of Scope (Future Work)

- Automated anomaly detection
- External export/integrations
- Rich audit dashboards (belongs to reporting/analytics)
- Per-tenant configurable retention and redaction rules

---

# End of Document
