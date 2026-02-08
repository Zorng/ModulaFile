# Edge Case Contract — Identity/HR ↔ POS Operations Boundary

## Metadata
- **Contract Type**: Edge Cases (Boundary)
- **Scope**: Cross-context boundary between Identity/HR and POS Operations
- **Primary Audience**: Backend, Frontend, QA
- **Owner(s)**: Work Start/End Orchestration, Cash Session, Sale orchestration, Access Control, Audit
- **Last Updated**: 2026-02-08
- **Delivery Level**:
  - **March**: must be handled or explicitly degraded safely
  - **Later**: explicitly deferred

---

## Purpose
This document hardens the boundary where **human work lifecycle** meets **cash + device + order workflows**.
It prevents contradictions like “staff ended work while drawer is open” and clarifies what must be enforced vs what is auditable.

---

## Scope
### In-scope
- Attendance ↔ Cash Session dependency
- Branch context selection and switching
- `role_key` / membership changes mid-session
- Multi-device sessions safety behavior
- Sale finalization prerequisites and audit signals
- Offline reconciliation constraints (March-safe definition)

### Out-of-scope
- Full offline-first HR guarantees (beyond best effort)
- Operational notifications workflow (future)
- Subscription billing enforcement (future)

---

## Definitions / Legend
- **Owner**: where the decision belongs (domain/process)
- **March**: must be handled or clearly defined for March delivery
- **Signal**: audit/log/UX message expected

---

## Boundary Principles (Hard Rules)

1. **Work is not the same as Cash Session**
   - Attendance = “person working”
   - Cash Session = “cash responsibility state”

2. **You cannot erase responsibility by ending work**
   - END_WORK must not silently leave open cash responsibilities.

3. **Branch context is mandatory for POS actions**
   - Each action runs within exactly one branch context.

4. **Authorization is checked at critical moments**
   - Access Control must re-evaluate membership status, `role_key`, and branch assignment on sensitive actions.

---

## Edge Case Catalog

### EC-BND-01 — Ending Work While Cash Session Is Still Open
- **Scenario**: Staff tries END_WORK while they have an ACTIVE cash session.
- **Trigger**: END_WORK request.
- **Expected Behavior**:
  - Block END_WORK with message:
    > “You must close your cash session before ending work.”
- **Signal**: UI message + audit log
- **Owner**: Work End orchestration + Cash Session
- **March**: Yes

### EC-BND-02 — Cash Session Open Without Attendance
- **Scenario**: Cash session exists but attendance is missing (bug/offline/sync).
- **Trigger**: Cash session query detects no ACTIVE attendance for that staff+branch.
- **Expected Behavior**:
  - Do not auto-close cash session.
  - Flag INCONSISTENT_STATE for audit; allow manager correction later.
- **Signal**: audit flag
- **Owner**: Cash Session + Audit + Attendance correction process
- **March**: Yes (with safe degradation)

### EC-BND-03 — Attendance Active Without Cash Session
- **Scenario**: Staff is checked-in but has no cash session.
- **Trigger**: Staff enters POS.
- **Expected Behavior**:
  - Allowed.
  - Block drawer-required actions until cash session is opened.
- **Signal**: optional UI hint
- **Owner**: POS UI + Access Control + Cash Session
- **March**: Yes

### EC-BND-04 — Switching Branch Mid-Shift
- **Scenario**: Staff checked-in at Branch A attempts to operate Branch B.
- **Trigger**: Branch-scoped action at different branch.
- **Expected Behavior**:
  - Require END_WORK at Branch A before START_WORK at Branch B.
- **Signal**: UI message
- **Owner**: Work Start/End orchestration + Access Control
- **March**: Yes

### EC-BND-05 — Logged In on Two Devices at Once
- **Scenario**: Same staff identity uses two devices simultaneously.
- **Trigger**: concurrent sessions.
- **Expected Behavior**:
  - START_WORK is idempotent (single ACTIVE attendance).
  - Cash session is single-active per staff+branch.
  - Low-risk screens may still be usable.
- **Signal**: optional warning
- **Owner**: Authentication session mgmt + Attendance + Cash Session
- **March**: Yes (state safety)

### EC-BND-06 — Membership / Role Key Changes Mid-Shift
- **Scenario**: Membership is disabled/archived, or `role_key` changes while staff is working.
- **Trigger**: sensitive action (void, finalize sale, open drawer, etc.)
- **Expected Behavior**:
  - Access Control re-check denies action if current facts/policy no longer allow it.
  - UI shows “Your access has changed.”
- **Signal**: UI message + audit log
- **Owner**: Access Control + UX contract
- **March**: Yes

### EC-BND-07 — Staff Archived While Still Working
- **Scenario**: Owner archives staff who is checked-in and/or has open cash session.
- **Trigger**: archive attempt.
- **Expected Behavior**:
  - Block archive until responsibilities resolved (attendance ended, cash session closed).
- **Signal**: UI message + audit log
- **Owner**: Staff Management + Work Start/End orchestration + Cash Session
- **March**: Strongly recommended

### EC-BND-08 — Finalize Sale Without Attendance
- **Scenario**: Staff finalizes a sale while not checked-in.
- **Trigger**: finalize sale.
- **Expected Behavior**:
  - Allow finalize sale if authenticated + authorized + branch context is valid.
  - Flag audit signal “sale_processed_without_attendance”.
- **Signal**: audit flag
- **Owner**: Sale orchestration + Audit
- **March**: Yes (resilience)

### EC-BND-09 — Cash Movement vs Attendance Timing
- **Scenario**: Sale records cash movement but attendance ends immediately after.
- **Trigger**: finalize sale then end work.
- **Expected Behavior**:
  - Financial correctness is preserved via Cash Session/Sale records.
  - Attendance is context, not source of truth for cash ledger.
- **Signal**: none required
- **Owner**: Cash Session + Sale
- **March**: Yes

### EC-BND-10 — Offline Mode and Reconciliation Constraints
- **Scenario**: Device works offline; HR + POS events sync later.
- **Trigger**: offline sync replay.
- **Expected Behavior**:
  - Events are idempotent + order-tolerant where feasible.
  - Conflicts must be auditable.
  - HR features may be “best effort” in March; correctness prioritized for cash/sales.
- **Signal**: audit conflict entries
- **Owner**: Offline Sync + Audit
- **March**: Partial (declare constraints)

---

## Decisions to Lock for March

1) **END_WORK blocked if cash session open** → Block (default)
2) **Finalize sale requires attendance** → No hard block; audit flag
3) **Archiving staff with active responsibilities** → Block archive
4) **Branch switching mid-shift** → Require end/start

---

## Related Processes

- `BusinessLogic/4_process/20_OrgAccount/10_tenant_membership_administration_process.md`
- `BusinessLogic/4_process/20_IdentityAccess/10_identity_activation_recovery_orchestration.md`
- `BusinessLogic/4_process/10_WorkForce/10_work_start_end_orchestration.md`
- `BusinessLogic/4_process/10_WorkForce/20_work_end_orchestrastion.md`

---

## Summary
- This contract defines the minimum “safety laws” across HR and POS.
- It prevents silent responsibility loss and clarifies audit-first degradation for March.
- It should be referenced by Work Start/End orchestration and Sale finalization orchestration.

_End of boundary edge case contract_
