# OperationalNotification Module — Platform System (Capstone 1)

**Version:** 1.0  
**Status:** Draft (Baseline locked: in-app only; signals not tasks)  
**Module Type:** Platform System (Core Capability)  
**Depends on:** Authentication & Authorization, Tenant & Branch Context, Access Control, Offline Sync (replay safety), Audit Logging (optional)  
**Related Modules/Processes:** Void Sale orchestration, Cash Session closeout, HR signals (future)

Authoritative references:
- Story: `BusinessLogic/1_stories/correcting_mistakes/responding_to_operational_notifications.md`
- Domain: `BusinessLogic/2_domain/60_PlatformSystems/operational_notification_domain.md`
- Edge cases: `BusinessLogic/3_contract/10_edgecases/operational_notification_edge_case_sweep.md`
- Emission process: `BusinessLogic/4_process/60_PlatformSystems/30_operational_notification_emission_process.md`

---

## 1. Purpose

OperationalNotification is a platform capability that emits **in-app notifications** when an operational state change requires:
- attention (someone should notice and potentially act),
- feedback (someone should know the outcome),
- awareness (FYI that something important happened),
- alert (exceptional condition worth reviewing).

OperationalNotification helps humans coordinate, but **does not participate in business logic**.

---

## 2. Scope & Boundaries

### In Scope (March Baseline)
- In-app inbox per user
- Unread badge/count
- Notifications emitted by operational workflows (Void, Cash Session closeout; more later)
- Role/responsibility-based recipients (approver pool, requester, oversight)
- Idempotent creation (safe under retries/offline replay)

### Explicitly Out of Scope
- Email/SMS/push delivery
- Escalation policies (“if not approved in 5 minutes…”)
- Assignment/claiming of tasks (“this manager owns it”)
- Workflow enforcement (notifications must not be required for correctness)
- Marketing/promotions/announcements/billing reminders

---

## 3. Principles (Non-Negotiable)

1. **Signals, not tasks**
2. **Best-effort**
3. **State is authority** (deep-linked actions re-check current state)
4. **Idempotent & replay-safe**
5. **Access-safe** (no cross-tenant/branch leakage)

---

## 4. Trigger + Recipient Model

### 4.1 Trigger Model

A valid trigger is:
- a business state transition (not a read),
- creates a waiting/awareness gap,
- does not affect correctness if missed.

### 4.2 Recipient Strategies

- **Approver pool**: all users who can perform the next action in the same `(tenant_id, branch_id)` context (resolved via Access Control).
- **Requester**: the actor who initiated the request.
- **Oversight roles**: Admin/Owner for alerts (rare).

Recipients are eligible viewers, not task owners.

---

## 5. Baseline Use Cases (Capstone 1)

### ON-01: Notify Approvers — Void Request Pending
**Trigger:** Sale transitions to `VOID_PENDING` (void requested)  
**Intent:** Attention  
**Recipients:** all users eligible for `sale.void.approve` in `(tenant_id, branch_id)` (typically Manager + Admin)  
**Deep Link:** void request detail / sale detail  
**Dedupe key:** `VOID_APPROVAL_NEEDED:{branch_id}:{sale_id}`

---

### ON-02: Notify Requester — Void Approved
**Trigger:** Sale transitions to `VOIDED` (after approval + effects complete)  
**Intent:** Feedback  
**Recipients:** the void requester  
**Deep Link:** sale detail / receipt view showing VOIDED  
**Dedupe key:** `VOID_APPROVED:{branch_id}:{sale_id}`

---

### ON-03: Notify Requester — Void Rejected
**Trigger:** void request rejected; Sale returns to `FINALIZED`  
**Intent:** Feedback  
**Recipients:** the void requester  
**Deep Link:** sale detail showing rejection reason (if provided)  
**Dedupe key:** `VOID_REJECTED:{branch_id}:{sale_id}`

---

### ON-04: Notify Managers — Cash Session Closed (Optional March)
**Trigger:** CashSession transitions to `CLOSED`  
**Intent:** Awareness  
**Recipients:** managerial oversight in branch (users eligible for `cashSession.z.view` in that branch)  
**Deep Link:** Z report / cash session summary  
**Dedupe key:** `CASH_SESSION_CLOSED:{branch_id}:{cash_session_id}`
**Notes (March):**
- Do not threshold “large variance” alerts.
- Include variance values in the notification body/payload so humans can evaluate.

---

## 6. Inbox Use Cases (Read/UX Behavior)

### UC-1: View Notification Inbox
**Actor:** any authenticated user  
**Preconditions:** user has access to tenant/branch contexts in which notifications exist  
**Behavior:**
- shows notifications visible to the actor
- supports unread/read presentation
- deep-link navigation into the subject detail

### UC-2: Mark Notification as Read
**Actor:** notification recipient  
**Behavior:**
- marking read updates recipient read state
- does not mutate the underlying business truth

---

## 7. Data Model (Suggested Minimal)

### operational_notifications
- id (uuid)
- tenant_id, branch_id
- type (enum/string)
- subject_type (sale, cash_session, ...)
- subject_id
- title, body (short)
- created_at
- dedupe_key (string)

### operational_notification_recipients
- notification_id
- recipient_user_id
- read_at (nullable)

Recommended uniqueness:
- unique `(tenant_id, dedupe_key)`
- unique `(notification_id, recipient_user_id)`

---

## 8. Idempotency & De-Duplication (Required)

Each notification type must generate a deterministic `dedupe_key`, for example:
- `VOID_APPROVAL_NEEDED:{branch_id}:{sale_id}`
- `VOID_APPROVED:{branch_id}:{sale_id}`
- `VOID_REJECTED:{branch_id}:{sale_id}`
- `CASH_SESSION_CLOSED:{branch_id}:{cash_session_id}`

This ensures safe behavior under retries, offline replay, or duplicated emissions.

---

## 9. Security & Access Control

- Recipients must be computed using Access Control + branch access rules.
- Users can only query notifications in tenants/branches they can access.
- Notifications must not leak sensitive information across tenants.
- Deep-linked actions must re-check authorization and current state.

---

## 10. Future Extensions (Explicit, Not March)

- HR operational signals (late check-in, missing check-out, correction requested)
- Notification preferences and per-type muting
- External delivery channels (push/email/SMS)
- Escalation/assignment/task management (separate domain; not OperationalNotification)

---

## 11. Acceptance Criteria (March)

- Notifications do not duplicate under retries/offline replay (dedupe works).
- Business truth remains correct if notification emission fails (best-effort).
- Recipients are correct and access-safe (no cross-tenant/branch leakage).
- Deep-link actions remain safe under stale notifications (state re-check).
- Inbox supports unread/read and deep link navigation.
