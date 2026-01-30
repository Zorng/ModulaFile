# BusinessLogic Guide — How to Use This Knowledge Base

## Purpose

This guide explains **how developers (human or AI) should work with the BusinessLogic knowledge base**.

It answers practical questions such as:
- Where do I add or change a rule?
- When do I write a domain vs a process vs a modspec?
- How do I avoid introducing hidden coupling or logic leaks?
- How do I safely evolve the system?

This guide assumes you have already read `BusinessLogic/README.md`.

---

## Mental Model (Read This First)

Modula’s business logic is structured into **three explicit layers**:

1. **Domain** — what must always be true  
2. **Process** — how domains cooperate  
3. **ModSpec** — how behavior is implemented  

If you remember only one rule:

> **Never skip a layer.**

Most design problems come from jumping directly to ModSpec.

---

## The Golden Rules

1. **Do not invent business rules in ModSpecs**
2. **Do not orchestrate cross-domain behavior inside a domain**
3. **Do not hide workflows inside code**
4. **If it spans domains, it must be a process**
5. **If it changes truth, it must be auditable**

Violating these rules creates “invisible logic” — the hardest bugs to fix later.

---

## When to Write or Update a Domain (`domain/`)

### Write or patch a domain document when:
- you introduce a new business concept
- you discover unclear ownership (“who decides this?”)
- you find hidden assumptions (“this works only if…”)
- you need to explain *why* something exists (user reality)
- you are changing invariants or lifecycle rules

### Domains must define:
- ownership and boundaries
- ubiquitous language
- invariants (non-negotiable rules)
- what is explicitly out of scope
- **why the model exists** (context matters)

### Domains must NOT:
- call other domains
- describe step-by-step workflows
- describe APIs or endpoints
- include implementation details

Example:
- Inventory domain defines ledger + projection
- Menu domain defines composition + tracking mode
- Discount domain defines eligibility + stacking rules

---

## When to Write or Update a Process (`process/`)

### Write or patch a process document when:
- behavior spans **more than one domain**
- timing or sequencing matters
- idempotency or retries are required
- compensating actions exist (reversal, adjustment)
- an event triggers multiple consequences

### Processes must define:
- trigger
- participating domains
- authoritative rules
- step ordering
- failure modes
- idempotency guarantees

### Processes must NOT:
- redefine domain invariants
- store long-term state
- compute domain-internal logic

Examples:
- Finalize Order
- Stock Deduction on Sale Finalization
- Inventory Reversal on Sale Void

If a rule involves “after X, then Y happens in another module”, it belongs here.

---

## When to Write or Update a ModSpec (`modSpec/`)

### Write or patch a modspec when:
- a domain or process is stable enough to implement
- UI or backend behavior must be clarified
- acceptance criteria are needed
- APIs or commands must be explicit

### ModSpecs must:
- follow the domain model exactly
- reference relevant processes
- define self-contained vs cross-module behavior
- specify what backend and frontend must do
- include acceptance criteria

### ModSpecs must NOT:
- introduce new rules
- redefine domain concepts
- contain “magic behavior”

If something feels unclear while writing a modspec:
👉 stop and patch the **domain or process first**.

---

## How to Evolve the System Safely

When introducing a change, follow this order:

1. **Explain the problem in real-world terms**
2. **Patch or add a domain document**
3. **Add or update a process if cross-domain**
4. **Patch the affected modspecs**
5. **Archive superseded documents**

Never reverse this order.

---

## Handling Ambiguity (Required Practice)

If something is unclear, do not guess silently.

Instead, document it:

### Assumption Template
- **Assumption:** what you are assuming
- **Reason:** why the docs are unclear
- **Impact:** what breaks if wrong
- **Decision Needed:** yes/no

This applies especially to AI-assisted development.

---

## Archived Documents (`_archived/`)

Archived files:
- preserve historical thinking
- explain why decisions changed
- support academic reflection

They are **not authoritative**.

When patching a document:
- move the old version to `_archived`
- create a new file with a clear name
- do not edit history in place

---

## Relationship to Playbooks and Templates

- **Playbooks** explain *how to think* (modeling philosophy)
- **Templates** explain *how to write*
- **BusinessLogic** explains *what the system is*

If you are unsure:
- read the playbook
- use the template
- then write or patch BusinessLogic docs

---

## Common Smells to Watch For

If you see any of these, stop and re-evaluate:
- “if feature flag enabled, then business rule applies”
- “this module just knows about the other one”
- “we’ll handle this in code”
- “this is easier to implement here”
- “this only happens sometimes, don’t worry”

These are signs the logic belongs in a **domain or process**, not a modspec.

---

## Final Reminder

> **This knowledge base is not documentation for code.  
> It is the contract that code must obey.**

Clarity here prevents complexity later.