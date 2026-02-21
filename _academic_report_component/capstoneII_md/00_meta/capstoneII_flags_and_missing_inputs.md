# Capstone II Draft — Flags & Missing Inputs Tracker

Purpose:
- Collect all places where the Capstone II draft needs **real-world input** (you, frontend, backend) that the KB cannot supply.
- Avoid inventing facts. We flag them explicitly and resolve them later.

How to use:
- When writing chapter markdown, insert `TODO_CAPSTONE2(<FLAG_ID>): <short note>` inline.
- Add a matching entry below with owner + required details.

Status legend:
- `OPEN` = missing info
- `IN_PROGRESS` = being collected
- `RESOLVED` = can be written into the draft

---

## Flag Index

| Flag ID | Owner | Category | Where (file / section) | What’s Missing | Status |
|---|---|---|---|---|---|
| FLAG-PM-01 | You | PM | `_academic_report_component/capstoneII_md/01_introduction/1.md` | Capstone II start/end dates + key milestones (midterm checkpoint, pilot target dates) | OPEN |

---

## Categories (Guidance)

- `PM` Project management (timeline, coordination, risks, adaptations)
- `FE` Frontend implementation status + screenshots/figures
- `BE` Backend implementation status + endpoints/DB design notes
- `INT` Integration status (FE↔BE; what’s wired vs stubbed)
- `OPS` Pilot / real-world constraints (devices, printers, KHQR ops, rollout)
- `EVAL` Evaluation/testing results (performance, usability, reliability)
- `FIG` Figures (architecture diagrams, screenshots, tables)
- `REF` References/bibliography updates

---

## OPEN Flags (To Be Filled As We Draft)

### PM — Timeline and Work Coordination

- **FLAG-PM-01** — Capstone II timeline dates and milestones (start/end, midterm checkpoint date, pilot date constraints).
- **FLAG-PM-02** — What changed vs Capstone I plan (scope cuts, reprioritization) and why.
- **FLAG-PM-03** — Team structure and division of responsibilities in Capstone II (who owns FE/BE/KB/spec).

### FE — Frontend Progress and Figures

- **FLAG-FE-01** — List of implemented screens/workflows in `/v0` frontend (and what’s stubbed).
- **FLAG-FE-02** — Screenshots/figures to include (naming + source of truth + final selection).
- **FLAG-FE-03** — UX deltas vs Capstone I prototype (what was improved, removed, deferred).

### BE — Backend Progress and Figures

- **FLAG-BE-01** — Implemented modules/services in `/v0` backend (what’s complete vs partial).
- **FLAG-BE-02** — DB schema / storage approach summary to present academically (what you can safely disclose).
- **FLAG-BE-03** — Sync interfaces implemented (push/pull) and what is still pending.
- **FLAG-BE-04** — Outbox usage: where it is implemented today and where it is planned.

### INT — Integration Status (FE↔BE)

- **FLAG-INT-01** — What is already integrated end-to-end (demoable), and what is not integrated yet.
- **FLAG-INT-02** — Known integration blockers (missing endpoints, auth wiring, environment config).

### OPS — Pilot Constraints / Hardware

- **FLAG-OPS-01** — Hardware reality for pilot (receipt printer, kitchen sticker printer): models, connection method, and what is in/out of scope for Capstone II draft.
- **FLAG-OPS-02** — KHQR/Bakong environment choice (SIT vs production token reality) and how you will safely test without harming real money flows.

### EVAL — Testing and Evaluation

- **FLAG-EVAL-01** — What evaluation you will include (manual testing, unit tests, integration tests, user feedback) and the concrete results/metrics.

### FIG — Architecture Diagrams

- **FLAG-FIG-01** — `/v0` architecture diagram: components + data flows (offline sync push/pull, webhook ingestion, outbox, job scheduler).
- **FLAG-FIG-02** — Prototype vs `/v0` comparison figure/table (what dimensions you want to show).

### REF — References

- **FLAG-REF-01** — New references to add for Capstone II (offline-first replication, outbox pattern, idempotency, modular architecture), and which ones to keep from Capstone I.
