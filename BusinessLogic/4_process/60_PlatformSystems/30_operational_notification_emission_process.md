# 30 — Operational Notification Emission (In-App Signals)

## Purpose

This process defines how the system emits **in-app operational notifications** when an operational state transition creates:
- attention need (waiting-for-human),
- outcome feedback,
- awareness,
- or alert conditions.

Operational notifications are **signals, not tasks**.
They must not be a correctness dependency for business truth.

---

## Story Traceability

Derived primarily from:
- `BusinessLogic/1_stories/correcting_mistakes/responding_to_operational_notifications.md`
- `BusinessLogic/1_stories/correcting_mistakes/voiding_a_sale.md`
- `BusinessLogic/1_stories/handling_money/closing_cash_drawer.md`

---

## Domains Involved

- Operational Notification (create inbox items + read state)
- Access Control (recipient eligibility + deep-link safety)
- Offline Sync (replay-safe idempotency)
- Audit (optional observational logging)

Source domains that trigger notifications (baseline):
- Sale/Order + Void workflows
- Cash Session

---

## When This Process Runs

Triggered after a **business state transition** has been persisted by a source workflow.

This process is best-effort:
- source workflows must not fail solely because notification emission fails.

---

## Inputs (Abstract)

Each emission attempt provides:
- `tenant_id`
- `branch_id` (March baseline: always required)
- `type` (notification type)
- `subject_type`, `subject_id`
- `dedupe_key` (deterministic)
- `payload` (small metadata needed for title/body + deep link)

---

## Baseline Notification Types (Capstone 1)

### ON-01: Void Request Pending (Notify Approvers)
- **Trigger**: Void request created with `status = PENDING` (Sale transitions to `VOID_PENDING`)
- **Recipients**: approver pool for `sale.void.approve` in `(tenant_id, branch_id)` (typically Manager + Admin)
- **Deep Link**: sale/void request detail
- **Dedupe key**: `VOID_APPROVAL_NEEDED:{branch_id}:{sale_id}`

### ON-02: Void Approved (Notify Requester)
- **Trigger**: Void request approved and Sale transitions to `VOIDED` (after effects complete)
- **Recipients**: void requester (actor who initiated the void request)
- **Deep Link**: sale detail / receipt view showing VOIDED
- **Dedupe key**: `VOID_APPROVED:{branch_id}:{sale_id}`

### ON-03: Void Rejected (Notify Requester)
- **Trigger**: void request rejected; Sale returns to `FINALIZED`
- **Recipients**: void requester
- **Deep Link**: sale detail showing rejection reason (if provided)
- **Dedupe key**: `VOID_REJECTED:{branch_id}:{sale_id}`

### ON-04: Cash Session Closed (Awareness, Optional March)
- **Trigger**: CashSession transitions to `CLOSED`
- **Recipients**: managerial oversight in branch (users eligible for `cashSession.z.view` in that branch)
- **Deep Link**: cash session closeout summary (Z artifact)
- **Dedupe key**: `CASH_SESSION_CLOSED:{branch_id}:{cash_session_id}`

March note:
- For March, do not introduce a “large variance” alert threshold.
- Include variance values in ON-04 payload/body so humans can evaluate.

---

## Orchestration Steps

### Step 1 — Validate trigger shape (fail closed)

- Require `tenant_id`, `branch_id`, `type`, `subject_type`, `subject_id`, and `dedupe_key`.
- Ensure `dedupe_key` is deterministic and includes subject identity (avoid collisions).

---

### Step 2 — Compute recipients (responsibility + access-safe)

Recipient strategies:
- **Approver pool**: all users eligible to perform the next action (resolved via Access Control role policy + branch access).
- **Requester**: the actor who initiated the request.
- **Oversight**: Admin/Owner roles with branch access (used sparingly for alerts).

Rules:
- Recipients must be scoped strictly within `(tenant_id, branch_id)`.
- Branch access is mandatory for recipients (no implicit “owner sees everything” bypass in March).

---

### Step 3 — Create notification idempotently

- Insert `operational_notifications` with `(tenant_id, dedupe_key)` uniqueness.
- On conflict (already exists), treat as success (idempotent).

---

### Step 4 — Attach recipients idempotently

- Insert recipient rows for each computed recipient.
- Enforce uniqueness per `(notification_id, recipient_user_id)` to avoid duplicates under retries.

---

### Step 5 — Deliver via in-app inbox (March baseline)

- Notification becomes visible in the recipient’s inbox.
- Unread badge/count increases for unread items.

No push/email/SMS channels exist in March.

---

## Failure / Degradation Rules (March)

- Notification emission is best-effort: failures do not rollback business truth.
- Offline replay must not duplicate notifications (dedupe key).
- Deep-linked actions must re-check current state (“state is authority”).

---

## Related Contracts

- `BusinessLogic/3_contract/10_edgecases/operational_notification_edge_case_sweep.md`
