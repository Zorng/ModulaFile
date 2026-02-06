# ModSpec (modspec_new) Versions

This file records the **Version** (and current **Status**, when present) declared in each document under `modspec/`.

Notes:
- Some specs contain multiple embedded headers (e.g., patched versions appended). For those files, the table uses the **latest** `**Version:**` found in the file.

| Reading order | Spec | Version | Status |
| --- | --- | --- | --- |
| 0 | `backend_guide.md` | — | — |
| 1 | `auth_module.md` | 1.4 | Revised (Supports provisioned account activation; aligns credential ownership with Staff onboarding; preserves multi-tenant per account) |
| 2 | `tenant_module.md` | 1.1 | Updated (Branch creation clarified; aligned with system-driven provisioning) |
| — | `branch_module.md` | 1.1 | Revised (Branch provisioning is system-driven) |
| 3 | `policy_module.md` | 1.3 | Patched (Branch-scoped policies + cash session requirement is product-enforced) |
| 4 | `offlineSync_module.md` | 1.2 | Patched (Cash sessions required for sales; frozen-branch enforcement) |
| 5 | `audit_module.md` | 1.1 | Patched (Branch lifecycle + frozen-branch denials added) |
| 6 | `cashSession_module.md` | 1.7 | Patched (Cash sessions required for sales; branch system-provisioning alignment) |
| 7 | `sale_module.md` | 2.4 | Patched (Cash session required for cart/sales; removes optional “no session” path for Capstone 1 build) |
| 8 | `discount_module.md` | 1.2 | Revised (Branch-scoped discounts clarified; stacking preserved; sale lock-in enforced) |
| 9 | `inventory_module.md` | 1.6 | Patched (Aligns with Sale integrity guarantees; removes optional cash-session wording) |
| 10 | `menu_module.md` | 1.2 | Revised & Approved |
| 11 | `staffAttendance_module.md` | 1.4 | Patched (Shift schedules added in Staff Management; attendance shift evaluation staged) |
| 12 | `staffManagement_module.md` | 1.4 | Patched (Owner-provisioned onboarding; user-owned credentials; aligned with Branch creation model + Capstone 1 boundaries) |
| 13 | `receipt_module.md` | 1.0 | Defined (Aligned with Capstone 1 boundaries) |
| 14 | `report_module.md` | 1.3 | Patched (Demo scope: Cash X + Z reports; access rules aligned to implementation) |
| — | `temp.md` | — | — |
