# Story Coverage Map — Cash Session (Handling Money)

**Purpose:** Track which human-centered stories exist for Cash Session cash handling, and which technical use cases they cover.  
**Scope:** CashSession domain + CashSession modspec (Capstone 1).  
**Rule:** Stories are grouped by **human operational context**, not by modules.

---

## 1) Source Artifacts (Authoritative)

- **Domain:** `BusinessLogic/2_domain/30_POSOperation/cashSession_domain.md`
- **ModSpec:** `BusinessLogic/5_modSpec/30_POSOperation/cashSession_module_patched_v2.md`

---

## 2) Operational Context (Where These Stories Live)

Recommended location (human context):
- `BusinessLogic/1_stories/handling_money/`

This context is chosen because Cash Session is experienced by staff as **cash drawer responsibility** across a shift.

---

## 3) Story Set (Human Situations)

These are the **minimum** stories that cover the module’s use cases without exploding into “one story per UC”.

1. **Opening the Cash Drawer**  
   File: `BusinessLogic/1_stories/handling_money/opening_cash_drawer.md`  
   Human situation: starting a shift/accountability window with an opening float.

2. **Handling Cash During the Shift**  
   File: `BusinessLogic/1_stories/handling_money/handling_cash_during_session.md`  
   Human situation: day-to-day cash movements, corrections, and mid-shift visibility.

3. **Closing the Cash Drawer**  
   File: `BusinessLogic/1_stories/handling_money/closing_cash_drawer.md`  
   Human situation: end-of-shift reconciliation, counted cash, variance, closure.

4. **Force Closing a Cash Session (Exception Handling)**  
   File: `BusinessLogic/1_stories/handling_money/force_close_cash_session.md`  
   Human situation: manager/admin needs to end a session despite imperfect conditions.

---

## 4) Coverage Mapping (Use Case → Story)

### Story 1 — Opening the Cash Drawer
Covers:
- **UC-1:** Start Cash Session (Open)

Notes:
- Includes the operational expectation: “no cash sales until drawer accountability begins.”

---

### Story 2 — Handling Cash During the Shift
Covers:
- **UC-2:** Continue Active Cash Session
- **UC-3:** Record Cash Tender From Sale (System)
- **UC-4:** Cash Paid In (Optional)
- **UC-5:** Paid Out (Policy-controlled)
- **UC-6:** Cash Refund (Approval-controlled)
- **UC-7:** Manual Cash Adjustment (Policy-controlled)
- **UC-10:** View X Report (Operational Snapshot)

Cross-module participation referenced (in story, at high level):
- **CM-1:** Finalize Sale → append SALE_IN cash movement (idempotent)
- Void/refund approval outcome → append REFUND_CASH / VOID cash out movement (idempotent)

Notes:
- This story should explain that “expected cash” is derived from an append-only movement history.
- It should *not* discuss storage, APIs, or projections in technical detail—only operational meaning.

---

### Story 3 — Closing the Cash Drawer
Covers:
- **UC-8:** Close Cash Session (Normal Close)
- **UC-11:** View Z Report (Session Close Summary)

Notes:
- Z report is treated as an operational artifact: “this is the close record,” not analytics.

---

### Story 4 — Force Closing a Cash Session (Exception Handling)
Covers:
- **UC-9:** Force Close Cash Session (Manager/Admin only)

Notes:
- This story should emphasize why force close exists, what it protects (history preserved), and what it cannot fix (human mistakes).

---

## 5) Coverage Summary (Quick Check)

| Use Case | Covered By Story |
|---|---|
| UC-1 Start Cash Session | Story 1 |
| UC-2 Continue Active Session | Story 2 |
| UC-3 Record Cash Tender From Sale | Story 2 |
| UC-4 Paid In | Story 2 |
| UC-5 Paid Out | Story 2 |
| UC-6 Cash Refund (Approval-controlled) | Story 2 |
| UC-7 Manual Cash Adjustment | Story 2 |
| UC-8 Close Cash Session | Story 3 |
| UC-9 Force Close | Story 4 |
| UC-10 View X Report | Story 2 |
| UC-11 View Z Report | Story 3 |
| CM-1 Finalize Sale → Cash Movement | Story 2 (reference) |
| CM-2 Reporting Read Contract | Not a story (non-human) — referenced only if needed in reporting stories |
  
---

## 6) Status Tracker (Fill As You Write)

| Story | File | Status | Notes |
|---|---|---|---|
| Opening the Cash Drawer | `BusinessLogic/1_stories/handling_money/opening_cash_drawer.md` |  ok | |
| Handling Cash During the Shift | `BusinessLogic/1_stories/handling_money/handling_cash_during_session.md` | ok | |
| Closing the Cash Drawer | `BusinessLogic/1_stories/handling_money/closing_cash_drawer.md` | ok| |
| Force Closing a Cash Session | `BusinessLogic/1_stories/handling_money/force_close_cash_session.md` | ok | |

Legend:
- ⬜ Not written
- 🟨 Drafted
- ✅ Final

---

## 7) Guardrails (To Prevent Story Drift)

- Do **not** write “one story per UC”. Keep stories tied to human situations.
- Stories may mention “System Areas Touched” (Sale / CashSession / Reporting) but must remain non-technical.
- If a new UC appears later, decide whether it:
  - fits into an existing story (most common), or
  - represents a new human situation (rare), then add a new story row.

---

# End of Map
