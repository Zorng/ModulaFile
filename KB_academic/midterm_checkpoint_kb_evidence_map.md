# Capstone Midterm Checkpoint — KB Evidence Map (Slides 1–3)

This artifact maps the midterm presentation flow to concrete Knowledge Base documents, so slides (1) Timeline Position, (2) Prototype vs `/v0`, and (3) Architecture Bridge can cite authoritative specs instead of “trust me”.

Scope note:
- Slides (4) Frontend progress and (5) Backend progress are implementation status updates and should be presented by those teams.
- This doc is intentionally implementation-neutral and uses KB paths as citations.

---

## Slide 1 — Timeline Position (Where We Are Now)

Goal:
- Show what milestone this checkpoint represents and which “big rocks” have been locked (decisions that reduce risk and enable integration).

Suggested “big rocks” to list (pick the ones that are true for your timeline):
- Multi-tenant SaaS identity with invite/accept membership model.
- Branch monetization model: paid branch activation as billable workspace.
- Subscription entitlements gating: enabled vs read-only behavior.
- Offline-first sync: durable queue + push replay + pull hydration feed.
- Idempotency as a platform gate for write safety.
- Outbox/eventual processing as a reliable way to execute cross-module side effects.
- Webhook gateway as a normalized, idempotent ingestion surface (payments).

KB citations (choose 3–6 on the slide; the rest can be “backup”):
- Multi-tenant membership model:
  - `BusinessLogic/2_domain/10_Identity&Authorization/tenant_membership_domain.md`
  - `BusinessLogic/4_process/20_OrgAccount/10_tenant_membership_administration_process.md`
- Branch activation monetization lock:
  - `BusinessLogic/5_modSpec/20_OrgAccount/branch_module.md`
  - `BusinessLogic/2_domain/60_PlatformSystems/subscription_entitlements_domain.md`
  - `BusinessLogic/4_process/60_PlatformSystems/90_first_branch_activation_orchestration.md`
  - `BusinessLogic/4_process/60_PlatformSystems/94_additional_branch_activation_orchestration.md`
- Offline-first sync lock (push + pull):
  - `BusinessLogic/2_domain/60_PlatformSystems/offline_sync_domain.md`
  - `BusinessLogic/5_modSpec/60_PlatformSystems/offlineSync_module.md`
  - `BusinessLogic/4_process/60_PlatformSystems/60_offline_operation_queue_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/70_offline_sync_replay_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/65_offline_sync_pull_hydration_process.md`
- Idempotency platform gate:
  - `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`
- Webhook payments ingestion (for KHQR + billing confirmations):
  - `BusinessLogic/5_modSpec/60_PlatformSystems/webhookGateway_module.md`
  - `BusinessLogic/4_process/60_PlatformSystems/96_webhook_ingestion_and_dispatch_process.md`
- Job scheduler (if you mention background processing):
  - `BusinessLogic/5_modSpec/60_PlatformSystems/jobScheduler_module.md`
  - `BusinessLogic/4_process/60_PlatformSystems/97_job_scheduler_tick_and_execution_process.md`

Speaker note:
- Keep Slide 1 short: 1 timeline graphic + 3–6 locked decisions.

---

## Slide 2 — Capstone 1 Prototype vs Current `/v0`

Goal:
- Demonstrate engineering maturity: identify “prototype smells” and show the architecture/spec decisions that address them.

Recommended format:
- Left column: prototype pain / smell (what breaks in real ops).
- Right column: `/v0` architectural fix (what is now specified and being built).

Prototype smells to consider (pick 4–6 max):
- “Offline” without convergence guarantees leads to drift between devices and server.
- Duplicate writes / retries cause double finalize, double cash movement, or double printing.
- Cross-module flows produce partial state (sale finalized but downstream effects missing).
- Ambiguous staff access lifecycle (disable/archive/delete drift) breaks auditability.
- Monetization model confusion (“branch slots” as reusable tokens) causes product drift.
- External payment confirmations (KHQR) require idempotent webhook ingestion and deterministic reconciliation.

KB citations for the “fix” side:
- Offline drift fix: authoritative hydration (pull) plus replay (push):
  - `BusinessLogic/2_domain/60_PlatformSystems/offline_sync_domain.md`
  - `BusinessLogic/4_process/60_PlatformSystems/65_offline_sync_pull_hydration_process.md`
  - `BusinessLogic/3_contract/10_edgecases/offline_sync_edge_case_sweep.md`
- Duplicate write fix: idempotency gate + idempotent downstream handlers:
  - `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`
  - `BusinessLogic/3_contract/10_edgecases/webhook_ingestion_edge_case_sweep.md`
- Partial state fix: explicit outbox/eventual processing allowance in POS core flows:
  - `BusinessLogic/4_process/30_POSOperation/10_finalize_sale_orch.md`
  - `BusinessLogic/4_process/30_POSOperation/13_stock_deduction_on_finalize_sale_process.md`
- Staff lifecycle fix: membership revoke preserves history (no delete/disable/archive drift):
  - `BusinessLogic/2_domain/10_Identity&Authorization/tenant_membership_domain.md`
  - `BusinessLogic/5_modSpec/30_HR/staffManagement_module.md`
  - `BusinessLogic/4_process/20_OrgAccount/10_tenant_membership_administration_process.md`
- Monetization semantics fix: paid activation, no reusable “slot” token:
  - `BusinessLogic/5_modSpec/20_OrgAccount/branch_module.md`
  - `BusinessLogic/3_contract/10_edgecases/subscription_billing_edge_case_sweep.md`

Speaker note:
- Make Slide 2 your strongest “engineering learning” slide.
- Explicitly say: prototype was correct for exploration; `/v0` is correct for real operations.

---

## Slide 3 — Architecture Bridge (Contracts That Enable Integration)

Goal:
- Show the minimal set of platform contracts that all modules align to.
- Explain why frontend and backend can now integrate systematically (not by ad-hoc endpoints).

Suggested “architecture bridge” bullets (keep it flat and concrete):
- Multi-tenant identity and membership is the root of “who can do what where”.
- Access Control is centralized; modules do not implement ad-hoc permission checks.
- Idempotency gate is mandatory for write paths (online) and replay paths (offline).
- Offline-first sync is push + pull:
  - push/replay guarantees “exactly once effect” under retries,
  - pull/hydration guarantees convergence without polling every module.
- Outbox-style transactional boundary is used for cross-module side effects (where applicable).
- Webhook gateway normalizes external payments into internal events safely and idempotently.

KB citations (you can put these as “Evidence” footer on Slide 3):
- Identity/membership:
  - `BusinessLogic/2_domain/10_Identity&Authorization/tenant_membership_domain.md`
- Access control:
  - `BusinessLogic/2_domain/10_Identity&Authorization/accessControl_domain_consistency_patched.md`
  - `BusinessLogic/5_modSpec/10_IdentityAccess/accessControl_module.md`
- Idempotency:
  - `BusinessLogic/4_process/60_PlatformSystems/80_idempotency_gate_process.md`
- Offline-first push + pull:
  - `BusinessLogic/5_modSpec/60_PlatformSystems/offlineSync_module.md`
  - `BusinessLogic/4_process/60_PlatformSystems/60_offline_operation_queue_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/65_offline_sync_pull_hydration_process.md`
  - `BusinessLogic/4_process/60_PlatformSystems/70_offline_sync_replay_process.md`
- Webhook gateway:
  - `BusinessLogic/5_modSpec/60_PlatformSystems/webhookGateway_module.md`
  - `BusinessLogic/4_process/60_PlatformSystems/96_webhook_ingestion_and_dispatch_process.md`
- Outbox references in POS flows (if you mention it here):
  - `BusinessLogic/4_process/30_POSOperation/10_finalize_sale_orch.md`

---

## Optional Backup Slides (If Advisor Asks “How Do You Know It’s Safe?”)

Backup A: Edge-case discipline
- `BusinessLogic/3_contract/10_edgecases/offline_sync_edge_case_sweep.md`
- `BusinessLogic/3_contract/10_edgecases/webhook_ingestion_edge_case_sweep.md`
- `BusinessLogic/3_contract/10_edgecases/subscription_billing_edge_case_sweep.md`

Backup B: Auditability / accountability
- `BusinessLogic/4_process/60_PlatformSystems/40_audit_event_recording_process.md`
- `BusinessLogic/5_modSpec/60_PlatformSystems/audit_module.md`
- `BusinessLogic/1_stories/saas_governance/reviewing_audit_logs_for_accountability.md`

