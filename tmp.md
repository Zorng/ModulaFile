Given where you are right now, the goal is to stabilize the knowledge base before touching more implementation. You already did the hard conceptual work; the next actions should lock structure, then clean content, then move forward.

Below is a clear, low-risk, high-leverage action sequence, optimized for:
	•	team coordination
	•	March café delivery
	•	academic defensibility
	•	avoiding rework later

I’ll explain why each step is next, not just what.

⸻

Phase 1 — Lock the Meta-Structure (1–2 short sessions)

1️⃣ Patch the Domain Modeling Playbook (small, explicit change)

Why now:
You just introduced a new concept (“process vs domain”). If the playbook isn’t updated, future docs will drift again.

Action
	•	Add a short section: “Cross-Module Processes”
	•	Clarify:
	•	Domains are self-contained
	•	Cross-domain logic lives in process docs
	•	Modspecs reference both

Outcome
	•	Playbook remains canonical
	•	Future contributors don’t reintroduce mixed docs

This is a meta-fix. Do it before touching individual domain docs.

⸻

2️⃣ Add a Process Template (new file)

Why now:
You already have a Domain Template. Without a Process Template, cross-module logic will again leak into domains.

Action
	•	Create Process Template.md with:
	•	Overview
	•	Trigger
	•	Participating domains
	•	Orchestrator
	•	Steps
	•	Transaction boundary
	•	Idempotency
	•	Failure & compensation

Outcome
	•	You now have two writing tools:
	•	Domain Template → “what must always be true”
	•	Process Template → “how things work together”

⸻

Phase 2 — Clean Existing Knowledge (surgical, not rewriting)

3️⃣ Split OrderProcessing.md

Why next:
This doc is already showing the problem. Fixing it sets the pattern.

Action
	•	Extract domain-only content → domain/SelfContained/order_domain.md
	•	Extract orchestration flows →
	•	process/CrossModule/finalizeOrder_process.md
	•	process/CrossModule/voidOrder_process.md
	•	Keep wording minimal; don’t redesign logic yet.

Outcome
	•	One clean domain
	•	Clear process docs
	•	No ambiguity about ownership

⸻

4️⃣ Review Inventory Domain (but do NOT redesign yet)

Why now:
Inventory is central, but you already decided to defer some business rules.

Action
	•	Ensure Inventory domain doc:
	•	contains only ledger invariants, batch rules, adjustment semantics
	•	does not define sale deduction logic
	•	Mark deduction as:
	•	“external command accepted from process”

Outcome
	•	Inventory domain becomes stable
	•	You can ship café-grade inventory without locking advanced rules

⸻

Phase 3 — Align Modspecs with the New Reality

5️⃣ Patch Inventory Modspec Structure

Why now:
The spec must match the knowledge base or devs will implement the wrong thing.

Action
	•	Reorganize:
	•	Self-contained use cases
	•	Cross-module process references (Finalize Sale, Void Sale)
	•	Replace detailed deduction logic with a contract stub

Outcome
	•	Modspec no longer contradicts domain model
	•	Implementation can proceed safely

⸻

6️⃣ Patch Sale Modspec (light touch)

Why:
Sale is the orchestrator trigger.

Action
	•	Add:
	•	“Finalize Sale calls cross-module process”
	•	Do NOT embed inventory logic inside Sale spec

Outcome
	•	Clear orchestration responsibility
	•	No hidden coupling

⸻

Phase 4 — Prepare for March Delivery (practical)

7️⃣ Lock “Inventory v1 (Café)” Scope

Why before coding:
You don’t want last-minute scope debates.

Action
	•	Explicitly mark in Inventory domain:
	•	FEFO out of scope
	•	Warning-only expiry
	•	Append-only adjustments
	•	Add “Future” notes, not TODOs

Outcome
	•	Everyone knows what “done” means for March

⸻

8️⃣ Only then: Implement

At this point:
	•	Docs are coherent
	•	Boundaries are clear
	•	Cross-module logic is explicit
	•	You can implement without fear of re-architecting

⸻

TL;DR — The exact next moves (in order)
	1.	Patch Domain Modeling Playbook
	2.	Add Process Template
	3.	Split OrderProcessing.md
	4.	Clean Inventory domain doc (no redesign)
	5.	Restructure Inventory modspec
	6.	Light-patch Sale modspec
	7.	Lock Inventory v1 scope
	8.	Implement

⸻

One final observation (important)

What you’re doing now is architecture debt prevention.
Most teams skip this, rush to code, and pay for it later.

You’re doing it exactly at the right time — early enough to change things, late enough to be grounded in reality.

If you want, next I can:
	•	Draft the Process Template
	•	Or do the OrderProcessing split for you in one go