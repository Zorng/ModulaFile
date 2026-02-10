# Fair-Use Limits Domain Model — Modula POS

## Domain Name
Fair-Use Limits (Safety Guard Rails)

## Domain Type
Platform / Supporting Domain (Anti-abuse + Resource Protection)

## Domain Group
60_PlatformSystems

## Status
Draft (March baseline: limits exist as safety caps; not monetization)

---

## Purpose

This domain defines **technical safety limits** that protect Modula from:
- accidental runaway usage (misconfigured client, infinite retry loops),
- malicious automation (bots creating extreme entity counts),
- and resource exhaustion that would degrade the service for everyone.

These limits are **not pricing**.
They are guard rails that keep the system healthy.

This domain exists so other modules can answer consistently:
- "Does this write exceed a safety quota?"
- "Is this actor rate-limited for this action?"
- "How do soft vs hard limits behave?"

---

## Boundary & Ownership

### What This Domain Owns
- Definitions of **soft vs hard** technical limits.
- A standard vocabulary for:
  - entity quotas (count-based caps),
  - rate limits (request-frequency caps),
  - and the expected denial reasons.
- Counting rules (what counts, what does not).

### What This Domain Does NOT Own
- Commercial packaging and module entitlement enforcement.
  - That is owned by `Subscription & Entitlements`.
- Authorization decisions (who is allowed).
  - That is owned by `Access Control`.
- Operational workflows (when a write happens).
  - That is owned by process orchestrations.

---

## Key Concepts (Ubiquitous Language)

### Fair-Use Quota (Count-Based)
A quota is a safety cap on how much "persistent system surface" a tenant can create.

Examples:
- menu items
- stock items
- staff profiles

Quotas exist to prevent pathological cases (e.g., 1,000,000 staff profiles).

### Operational Quotas (Module-Owned, Optional)
Some modules may also enforce **operational working-set limits** (for UX/performance),
for example "active menu items" or "active stock items".

Important:
- operational quotas are not pricing,
- operational quotas may allow archiving to "free active capacity",
- fair-use caps remain the platform safety backstop and must not be bypassable by archive loops.

### Rate Limit (Frequency-Based)
A rate limit bounds how often a specific actor/client can attempt an action.

Examples:
- "create menu item" repeated in a tight loop
- offline queue replay that floods the backend

Rate limits are scoped by one or more of:
- actor (`auth_account_id`)
- tenant
- branch
- device (if available)
- IP/network identity (if available)

### Soft Limit vs Hard Limit
- **Soft limit:** approaching a cap; may warn/telemetry but does not necessarily block.
- **Hard limit:** blocks the write (fail closed).

Important:
- Soft limit behavior is allowed to be "warn only" for March.
- Hard limit behavior must be enforced consistently (server-side).

---

## March Baseline Defaults (Locked)

These defaults are intentionally **generous for normal cafes** and exist only to stop anomalies/bots.

Rule:
- Soft limit = 80% of hard limit.

### Entity Hard Limits (Count-Based Quotas)

| Entity kind | Scope | Soft | Hard |
|---|---|---:|---:|
| `staff_profiles` | TENANT | 4,000 | 5,000 |
| `branch_assignments` | TENANT | 40,000 | 50,000 |
| `menu_items` | TENANT | 4,000 | 5,000 |
| `menu_categories` | TENANT | 160 | 200 |
| `modifier_groups` | TENANT | 400 | 500 |
| `modifier_options` | TENANT | 8,000 | 10,000 |
| `stock_items` | TENANT | 8,000 | 10,000 |
| `stock_categories` | TENANT | 400 | 500 |

Notes:
- These are safety caps, not plan tiers.
- Append-only ledgers are not hard-capped by total count (audit logs, inventory journal, cash movements). Control them via rate limits and storage lifecycle policy instead.

### Write Rate Limits (Safety, Not Pricing)

Rate limits are scoped by `(tenant_id, actor_id)` unless otherwise stated.

| Action kind | Scope | Limit |
|---|---|---:|
| create staff/menu/stock item | TENANT+ACTOR | 60 / minute |
| create restock batch | BRANCH+ACTOR | 30 / minute |
| inventory adjustment | BRANCH+ACTOR | 30 / minute |
| OTP send | ACTOR | 3 / 10 minutes |
| login attempts | ACTOR | 10 / 10 minutes (then lockout) |

---

## Invariants (Non-Negotiable)

- INV-FU-1: Fair-use limits must never be framed as monetization.
- INV-FU-2: Hard limits block writes deterministically (no partial writes).
- INV-FU-3: Replays and retries must not consume quota twice:
  - idempotent requests that resolve to "already done" must not increment counts.
- INV-FU-4: Rate limiting must not break idempotency:
  - if a request is a duplicate of an already-applied idempotent write, it must be allowed to return the same success result.
- INV-FU-5: Fair-use enforcement is independent of module entitlements:
  - entitlements may block a write earlier, but quotas are not a substitute for entitlements.

---

## Counting Rules (Safety-First)

This domain standardizes how quotas are counted.

### Counted vs Not Counted
- Quotas should count **total persisted records** for an entity kind, not only "active".
  - Reason: otherwise a bot can create and archive endlessly.
- System-derived pseudo-entities (e.g., "Uncategorized") must not count toward quotas.

Note:
- If a module also defines an operational quota, it may count only "active" entities.
- Fair-use caps still count total persisted records.

### Scope
Each quota is defined with an explicit scope:
- `TENANT` scoped (most entity catalogs)
- `BRANCH` scoped (rare; used only when an entity is inherently per-branch)

---

## Canonical Outputs (Reason Codes)

When a write is blocked by fair-use enforcement, the system returns a denial reason code:

- `FAIRUSE_HARD_LIMIT_EXCEEDED`
- `FAIRUSE_RATE_LIMITED`

Soft-limit warnings are observational, not authorization:
- `FAIRUSE_SOFT_LIMIT_WARNING` (telemetry/UX hint; never a denial)

---

## Relationship to Other Domains

- **Subscription & Entitlements**
  - decides which modules are enabled / read-only / disabled-visible.
  - fair-use limits remain non-commercial safety caps.
- **Access Control**
  - decides ALLOW/DENY for actions.
  - fair-use is a post-authorization safety gate for writes.
- **Offline Sync + Idempotency**
  - idempotency must be resolved before fair-use increments.
  - offline replay should not create artificial limit failures if the operation was already applied.

---

## Process Reference (Enforcement)

Fair-use enforcement is expressed as a platform gate process:
- `BusinessLogic/4_process/60_PlatformSystems/85_fair_use_limit_gate_process.md`
