# ModSpec Versions

This file records the declared `**Version:**` and `**Status:**` headers for the canonical ModSpecs under `BusinessLogic/5_modSpec/`.

Notes:
- If a spec contains multiple embedded headers (patched versions appended), this table uses the latest `**Version:**` / `**Status:**` that appear in the file.
- Legacy specs live in `BusinessLogic/5_modSpec/_archived/`.

| Reading order | Spec | Version | Status |
| --- | --- | --- | --- |
| 1 | `10_IdentityAccess/authentication_module.md` | 2.0 | Patched (Renamed from `auth_module`; explicit AuthN vs AuthZ separation) |
| 2 | `10_IdentityAccess/accessControl_module.md` | 1.0 | Draft (Branch access is mandatory) |
| 3 | `20_OrgAccount/tenant_module.md` | 2.0 | Patched to support Access Control + future SaaS (without implementing billing yet) |
| 4 | `20_OrgAccount/branch_module.md` | 1.1 | Revised (Branch provisioning is system-driven) |
| 5 | `30_HR/staffManagement_module.md` | 3.1 | Patched (simplified onboarding; user-owned credentials; aligned with Authentication, Access Control, Tenant, and Concurrent Staff Licensing) |
| 6 | `30_HR/attendance_module.md` | 1.4 | Patched (Shift schedules added in Staff Management; attendance shift evaluation staged) |
| 7 | `50_PlatformSystems/policy_module.md` | 1.3 | Patched (Branch-scoped policies + cash session requirement is product-enforced) |
| 8 | `50_PlatformSystems/offlineSync_module.md` | 1.2 | Patched (Cash sessions required for sales; frozen-branch enforcement) |
| 9 | `50_PlatformSystems/audit_module.md` | 1.1 | Patched (Branch lifecycle + frozen-branch denials added) |
| 10 | `30_POSOperation/cashSession_module_patched_v2.md` | 1.3 (Patched) | Clarified to distinguish operational reconciliation (X/Z) from analytics reporting |
| 11 | `30_POSOperation/sale_module_patched.md` | 2.4 | Patched (Clarifies order lifecycle and draft persistence rules; preserves existing pricing/discount/cash-session integrity rules) |
| 12 | `30_POSOperation/discount_module_patched.md` | 1.3 (Patched) | Patched to align with `discount_domain.md` (eligibility + scope, no money math) and the Self-Contained vs Cross-Module structure |
| 13 | `30_POSOperation/inventory_module_patched.md` | 1.7 (Patched) | Patched to align with **Inventory Domain (ledger + projections)** and the new **Self-Contained vs Cross-Module** structure |
| 14 | `30_POSOperation/menu_module_patched.md` | 1.3 (Patched) | Patched to align with Menu Domain (composition + TRACKED components) |
| 15 | `30_POSOperation/receipt_module_patched.md` | 1.1 (Patched) | Clarified to align with snapshot-based Sale lifecycle and cross-module processes |
| 16 | `30_POSOperation/receipt_module.md` | 1.0 | Defined (Aligned with Capstone 1 boundaries) |
| 17 | `40_Reporting/report_module.md` | 1.3 | Patched (Demo scope: Cash X + Z reports; access rules aligned to implementation) |
