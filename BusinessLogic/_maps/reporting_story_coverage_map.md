# Story Coverage Map — Reporting (Management Insight)

**Purpose:**  
Identify the human-centered stories that justify Reporting features and map them to processes/modspecs, ensuring clear scope without "one story per UC".

**Scope:**  
- Reporting Domain (`BusinessLogic/2_domain/50_Reporting/reporting_domain.md`)  
- Reporting Module Spec (`BusinessLogic/5_modSpec/50_Reporting/report_module.md`)  

**Rule:**  
Stories remain grouped by **human operational context**, not by modules.

---

## 1) Source Artifacts (Authoritative)

- **Domain:** `BusinessLogic/2_domain/50_Reporting/reporting_domain.md`
- **ModSpec:** `BusinessLogic/5_modSpec/50_Reporting/report_module.md`
- **Edge cases:** `BusinessLogic/3_contract/10_edgecases/reporting_edge_case_sweep.md`
- **UX behavior:** `BusinessLogic/3_contract/20_ux_specs/management_reporting_ux_spec.md`

---

## 2) Operational Contexts (Human Reality)

Reporting is experienced by humans in two stable contexts:

1) **Management Review (Owner/Admin/Manager)**  
   Reviewing performance, patterns, and spending over time.

2) **Operational Closeout (Cash Accountability)**  
   Reviewing X/Z artifacts for a cash session (owned by Cash Session; surfaced in reporting UI).

---

## 3) Story Set (Human Situations)

### Context: Management Review

1. **Reviewing Sales Performance (Owner / Admin)**  
   File: `BusinessLogic/1_stories/selling_and_checkout/reviewing_sales_performance.md`

2. **Reviewing Attendance and Time Respecting**  
   File: `BusinessLogic/1_stories/handling_staff&attendance/review_attendance&time_respection.md`

3. **Reviewing Restock Spending (Owner / Admin)**  
   File: `BusinessLogic/1_stories/managing_stock/reviewing_restock_spending.md`

### Context: Operational Closeout (Referenced, Not Owned Here)

4. **Opening/Handling/Closing Cash Sessions (X/Z as operational artifacts)**  
   Primary files live under `BusinessLogic/1_stories/handling_money/` and are mapped in:  
   `BusinessLogic/_maps/cashSession_story_coverage_map.md`

---

## 4) Coverage Mapping (Story → Process → ModSpec)

### Story 1 — Reviewing Sales Performance
Primary process:
- `BusinessLogic/4_process/50_Reporting/10_sales_reporting_process.md`
Reporting modspec coverage:
- UC-1 Sales Summary
- UC-2 Sales Drill-down

---

### Story 2 — Reviewing Attendance and Time Respecting
Primary process:
- `BusinessLogic/4_process/10_WorkForce/40_attendance_report.md`
Reporting modspec coverage:
- UC-3 Attendance Insights

---

### Story 3 — Reviewing Restock Spending
Primary process:
- `BusinessLogic/4_process/50_Reporting/20_restock_spend_reporting_process.md`
Reporting modspec coverage:
- UC-5 Restock Spend Summary

---

### Story 4 — Cash Session Closeout (X/Z)
Primary owner stories:
- `BusinessLogic/1_stories/handling_money/*`
Reporting modspec coverage:
- UC-4 Cash Session Closeout (X/Z)

---

## 5) Guardrails (Important)

- Reporting must remain **read-only** and must not become a second source of truth.
- Inventory "reporting" for March remains primarily in Inventory views (on-hand, journal, restocks).
- Restock spending is allowed as management insight, but inventory valuation/COGS requires explicit later design.

---

## 6) Status Tracker (Optional)

| Story | File | Status |
|---|---|---|
| Reviewing Sales Performance | `BusinessLogic/1_stories/selling_and_checkout/reviewing_sales_performance.md` | ok |
| Reviewing Attendance and Time Respecting | `BusinessLogic/1_stories/handling_staff&attendance/review_attendance&time_respection.md` | ok |
| Reviewing Restock Spending | `BusinessLogic/1_stories/managing_stock/reviewing_restock_spending.md` | ok |

---

# End of Map

