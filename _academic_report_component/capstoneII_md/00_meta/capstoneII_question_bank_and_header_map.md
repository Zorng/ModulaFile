# Capstone II Draft — Question Bank and Header Map

Purpose:
- Keep implementation and deployment questions in one stable place while we draft the report header by header.
- Avoid losing useful questions across chat turns.
- Make it clear **who** should answer **what**, and **when** we need the answer.

How to use:
- When we start a chapter/header, check the relevant section below first.
- Ask only the questions mapped to that header unless we truly need cross-chapter facts early.
- When an answer is received, write it into the draft and mark the question `ANSWERED`.
- If the answer creates a figure/table need, also update `_academic_report_component/capstoneII_md/00_meta/capstoneII_flags_and_missing_inputs.md`.

Status legend:
- `OPEN` = not yet asked
- `ASKED` = sent to owner, waiting for answer
- `ANSWERED` = answer received
- `NOT_NEEDED` = no longer needed for final draft

---

## Quick Header Map

| Chapter | Header | Main Missing Inputs | Ask From | Ask Timing |
|---|---|---|---|---|
| 1 | Introduction | Capstone II dates, milestone framing | You | Before finalizing Ch.1 |
| 2 | Presentation of Project | scope changes, current product positioning, team roles | You | Before finalizing Ch.2 |
| 3 | Literature Review | citations and reference list updates | You | During Ch.3 polishing |
| 4 | Project Analysis and Concepts | prototype vs `/v0`, updated diagrams, architectural rationale | You + FE + BE | Before finalizing Ch.4 |
| 5 | Detail Concepts | infrastructure choices, hosting, OTP, Bakong billing path, hardware plan | You + FE + BE | Before or during Ch.5 |
| 6 | Implementation | actual FE/BE stacks, implemented flows, integration status, staging deployment, screenshots | FE + BE + You | Required before finalizing Ch.6 |
| 7 | Conclusion | evaluation results, actual implemented status, staging readiness, remaining risks | You + FE + BE | After Ch.6 is mostly complete |

---

## Question Bank

### Questions for You

| QID | Chapters | Ask When | Question | Status |
|---|---|---|---|---|
| Q-YOU-01 | 1, 7 | Early | What are the exact Capstone II start date, current checkpoint date, and target submission date? | OPEN |
| Q-YOU-02 | 1, 2 | Early | What milestone dates should appear in the report timeline (midterm, staging readiness, pilot target, final defense)? | OPEN |
| Q-YOU-03 | 2 | Early | What changed from the Capstone I plan to Capstone II in terms of scope and priorities? | ANSWERED |
| Q-YOU-04 | 2 | Early | How are responsibilities divided now across you, frontend team, backend team, and KB/spec work? | NOT_NEEDED |
| Q-YOU-05 | 2, 4 | During Ch.4 | What are the most important design smells or architectural weaknesses in the Capstone I prototype that justified the `/v0` overhaul? | OPEN |
| Q-YOU-06 | 4 | During Ch.4 | What dimensions do you want in the prototype vs `/v0` comparison table (e.g. offline support, auth model, billing readiness, modularity, deployment)? | OPEN |
| Q-YOU-07 | 4, 5 | During Ch.4/5 | Which architecture decisions were most important from your perspective and should be emphasized to the advisor? | OPEN |
| Q-YOU-08 | 5 | During Ch.5 | Which deployment providers are already locked by your decision, and which are still tentative? | OPEN |
| Q-YOU-09 | 5 | During Ch.5 | What is the intended staging environment story that the report should tell: internal staging only, advisor demo staging, or pilot-prep staging? | OPEN |
| Q-YOU-10 | 5, 6 | During Ch.5/6 | What is the current real staging URL set that can be named in the report, if any? | OPEN |
| Q-YOU-11 | 5 | During Ch.5 | What should the report say about KHQR/Bakong usage in staging: real token, sandbox absence, and safety precautions? | OPEN |
| Q-YOU-12 | 5, 7 | During Ch.5/7 | What hardware is in scope for pilot/staging right now: receipt printer, kitchen sticker printer, anything else? | OPEN |
| Q-YOU-13 | 6, 7 | Late | Which features should be described as fully implemented, partially implemented, and deferred? | OPEN |
| Q-YOU-14 | 7 | Late | What risks or unfinished items do you want stated explicitly in the conclusion? | OPEN |
| Q-YOU-15 | 7 | Late | Do you want the report to explicitly mention pilot-cafe deployment and staging deployment as evidence of maturity? | OPEN |

#### Answer Notes

- `Q-YOU-03`
  - Capstone II was reprioritized based on evidence from the Capstone I prototype.
  - The project shifted from feature demonstration toward architectural correction, implementation hardening, integration readiness, and deployment-oriented refinement.
  - Key correction areas included:
    - multi-tenant authentication and membership flow
    - clearer account / tenant / branch workspace separation
    - offline-first synchronization and retry safety
    - stronger cross-module consistency
    - safer integration boundaries across POS, inventory, workforce, and platform services
  - Capstone II also introduced selective functional additions where they directly supported real operation:
    - notifications
    - real KHQR integration with backend verification
    - phone-based OTP onboarding
    - GPS-supported attendance evidence
    - automatic inventory deduction on finalize
    - pay-later / open-ticket support
    - preparatory billing / subscription work
  - Some intended billing implementation work remained partially deferred because the required effort exceeded the remaining time and team capacity.

- `Q-YOU-04`
  - Detailed Capstone II role allocation is maintained in the `.docx`.
  - The markdown chapter keeps team organization generic and avoids duplicating role-detail that is already handled in the document layout layer.

### Questions for Frontend Team

| QID | Chapters | Ask When | Question | Status |
|---|---|---|---|---|
| Q-FE-01 | 4, 6 | Before Ch.6 | What frontend framework, language, and major libraries are actually used in `/v0`? | OPEN |
| Q-FE-02 | 4, 6 | Before Ch.6 | What major user flows are already implemented in the frontend right now? | OPEN |
| Q-FE-03 | 6 | Before Ch.6 | Which frontend flows are fully integrated with backend, and which are still mocked, stubbed, or partial? | OPEN |
| Q-FE-04 | 4, 6 | Before Ch.6 | How is the 3-layer workspace model implemented in UI: account layer, tenant layer, branch layer? | OPEN |
| Q-FE-05 | 4, 6 | Before Ch.6 | On small screens, how are tenant portal and branch portal actually presented? | OPEN |
| Q-FE-06 | 4, 6 | Before Ch.6 | What responsive breakpoints or device targets are actually used? | OPEN |
| Q-FE-07 | 6 | Before Ch.6 | What client-side state management and routing approach are used? | OPEN |
| Q-FE-08 | 5, 6 | Before Ch.6 | How is offline-first behavior implemented on the client: local persistence, queueing, sync pull trigger, degraded states? | OPEN |
| Q-FE-09 | 6 | Before Ch.6 | What screens or user flows should be captured as figures/screenshots for the report? | OPEN |
| Q-FE-10 | 6 | Before Ch.6 | What is the staging frontend URL and which hosting provider is used? | OPEN |
| Q-FE-11 | 6 | Before Ch.6 | Which printer workflows are already available in frontend behavior: receipt print, kitchen ticket print, both, or partial? | OPEN |
| Q-FE-12 | 6, 7 | Late | What frontend parts are still incomplete, unstable, or risky? | OPEN |
| Q-FE-13 | 2, 4 | During Ch.2/4 | What are the biggest UX differences between the Capstone I prototype and the current `/v0` frontend? | ANSWERED |

#### Answer Notes

- `Q-FE-13`
  - Capstone I frontend behaved more like a feature showcase / prototype, while the current frontend is workflow-driven and closer to real POS operations.
  - The current UI is organized around explicit `account -> tenant -> branch` layers, with role-aware portals and branch-scoped workspaces.
  - Responsive behavior is now a first-class concern:
    - mobile uses portal + tabs
    - wide screens use navigation rail + side panels
  - Several operational screens were substantially improved for real usage, especially:
    - sale / cart
    - cash session
    - staff management
    - attendance
    - history / reconciliation surfaces
  - The current frontend is more backend-driven and aligned to real contracts:
    - policy
    - cash session
    - attendance
    - staff management
    - KHQR payment flow
  - UX is now more role-aware and scope-aware:
    - owner/admin vs cashier/manager
    - tenant-level vs branch-level actions
    - read-only vs actionable states
  - Some technical/internal language was replaced with clearer operational wording for users, such as emphasizing session, movement, and history surfaces instead of exposing raw internal terminology.

### Questions for Backend Team

| QID | Chapters | Ask When | Question | Status |
|---|---|---|---|---|
| Q-BE-01 | 4, 6 | Before Ch.6 | What backend framework, language, and major infrastructure libraries are actually used in `/v0`? | OPEN |
| Q-BE-02 | 6 | Before Ch.6 | What backend modules/services are fully implemented right now? | OPEN |
| Q-BE-03 | 6 | Before Ch.6 | Which flows are fully integrated end-to-end with frontend? | OPEN |
| Q-BE-04 | 5, 6 | Before Ch.6 | What deployment providers are actually used in staging for backend, database, object storage, and any queue/cache components? | OPEN |
| Q-BE-05 | 5, 6 | Before Ch.6 | What is the staging backend URL or API base URL? | OPEN |
| Q-BE-06 | 5, 6 | Before Ch.6 | Is OTP verification implemented in staging? If yes, through which provider? | OPEN |
| Q-BE-07 | 5, 6 | Before Ch.6 | What is the real status of KHQR/Bakong integration: implemented in staging, partial, or still isolated? | OPEN |
| Q-BE-08 | 5, 6 | Before Ch.6 | Which background jobs are already implemented and running? | OPEN |
| Q-BE-09 | 4, 6 | Before Ch.6 | Is the outbox pattern fully implemented? For which flows? | OPEN |
| Q-BE-10 | 4, 6 | Before Ch.6 | Which offline sync parts are implemented: push, pull hydration, SSE nudge, tombstones, cursor/checkpoint logic? | OPEN |
| Q-BE-11 | 4, 6 | Before Ch.6 | Which consistency mechanisms are implemented: idempotency, deduplication, transactional outbox, retry safety? | OPEN |
| Q-BE-12 | 6 | Before Ch.6 | What storage/data model facts are safe to disclose academically? | OPEN |
| Q-BE-13 | 6, 7 | Late | What observability/logging exists in staging right now? | OPEN |
| Q-BE-14 | 6, 7 | Late | What backend parts remain incomplete or high-risk? | OPEN |

---

## Header-by-Header Pull List

### Chapter 1 — Introduction

Ask first:
- `Q-YOU-01`
- `Q-YOU-02`

Optional later:
- `Q-YOU-15`

---

### Chapter 2 — Presentation of Project

Ask first:
- `Q-YOU-03`
- `Q-YOU-04`
- `Q-FE-13`

Useful supporting questions:
- `Q-YOU-05`
- `Q-YOU-13`

---

### Chapter 3 — Literature Review

Ask first:
- `Q-YOU-07` if you want emphasis guidance

Resolve through references/flags:
- bibliography updates in flags tracker

---

### Chapter 4 — Project Analysis and Concepts

Ask first:
- `Q-YOU-05`
- `Q-YOU-06`
- `Q-YOU-07`
- `Q-FE-01`
- `Q-FE-04`
- `Q-FE-05`
- `Q-FE-13`
- `Q-BE-01`
- `Q-BE-09`
- `Q-BE-10`
- `Q-BE-11`

Likely outputs:
- prototype vs `/v0` comparison table
- updated architecture rationale
- updated diagrams

---

### Chapter 5 — Detail Concepts

Ask first:
- `Q-YOU-08`
- `Q-YOU-09`
- `Q-YOU-11`
- `Q-YOU-12`
- `Q-FE-10`
- `Q-BE-04`
- `Q-BE-05`
- `Q-BE-06`
- `Q-BE-07`
- `Q-BE-08`

Likely outputs:
- infrastructure/deployment subsection
- payment flow clarification
- hardware/environment assumptions

---

### Chapter 6 — Implementation

Ask first:
- `Q-FE-01`
- `Q-FE-02`
- `Q-FE-03`
- `Q-FE-04`
- `Q-FE-05`
- `Q-FE-06`
- `Q-FE-07`
- `Q-FE-08`
- `Q-FE-09`
- `Q-FE-10`
- `Q-FE-11`
- `Q-BE-01`
- `Q-BE-02`
- `Q-BE-03`
- `Q-BE-04`
- `Q-BE-05`
- `Q-BE-08`
- `Q-BE-09`
- `Q-BE-10`
- `Q-BE-11`
- `Q-BE-12`
- `Q-YOU-10`
- `Q-YOU-13`

Likely outputs:
- actual stack write-up
- implementation status tables
- staging/deployment evidence
- screenshot/figure list

---

### Chapter 7 — Conclusion

Ask after Chapter 6 is mostly stable:
- `Q-YOU-13`
- `Q-YOU-14`
- `Q-YOU-15`
- `Q-FE-12`
- `Q-BE-13`
- `Q-BE-14`

Likely outputs:
- honest completion summary
- remaining work
- deployment/evaluation perspective

---

## Suggested Asking Strategy

Do not ask every question at once.

Recommended sequence:
1. Ask **you-only** questions needed for the chapter we are currently writing.
2. Ask **frontend/backend** only the questions needed for the next chapter we are about to finalize.
3. When answers come back, update:
   - the chapter markdown
   - the flags tracker
   - this question bank status

This keeps the report accurate without turning information gathering into a separate project.
