# Fair-Use Limits — Edge Case Sweep (Safety, Not Pricing)

## Purpose

This contract locks how Modula handles safety limits so modules do not drift into:
- ad-hoc "quota" behavior,
- monetization by hidden caps,
- or breaking offline replay/idempotency.

This contract is about **technical guard rails**, not product packaging.

---

## Definitions (Contract-Level)

- **Soft limit:** warn-only; never blocks writes in March baseline unless explicitly configured.
- **Hard limit:** blocks the write; deterministic.
- **Rate limit:** blocks repeated attempts within a time window; must not break idempotency.

Authoritative domain:
- `BusinessLogic/2_domain/60_PlatformSystems/fair_use_limits_domain.md`

Canonical enforcement process:
- `BusinessLogic/4_process/60_PlatformSystems/85_fair_use_limit_gate_process.md`

---

## Edge Cases (Locked)

### FU-EC-01 — Two admins create the same entity concurrently

**Scenario:** Two admins create many menu items simultaneously and the system crosses a hard quota boundary.

**Required behavior:**
- At most one request past the hard limit may succeed depending on timing, but the system must converge to:
  - count <= hard limit (no overshoot by unbounded races).
- Denials must be deterministic once hard limit is reached.

**Notes:**
- Implementation may require DB constraints/transactions; contract only requires the outcome.

---

### FU-EC-02 — Offline replay repeats a "create" operation

**Scenario:** Client replays queued operations after outage; the same "create stock item" is replayed multiple times.

**Required behavior:**
- Idempotent duplicates must return the same successful result and must not consume quota again.
- Rate limiting must not prevent the duplicate from returning its success result.

---

### FU-EC-03 — Bot tries to create extremely large entity counts

**Scenario:** A malicious client creates 1M staff profiles or menu items.

**Required behavior:**
- Hard limit blocks the write well before resource exhaustion.
- Denial reason must be `FAIRUSE_HARD_LIMIT_EXCEEDED` (not "upgrade plan").
- System remains operable for other tenants.

---

### FU-EC-04 — Hard limit is reached, then user archives items

**Scenario:** A tenant hits a hard quota, then archives/deactivates many entities and expects to create more.

**Required behavior (March baseline):**
- Fair-use caps count **total persisted records**, not only active.
- Archiving does not "free capacity" under fair-use safety caps.
- If a module also has an operational quota (active working-set limit), that module may allow archiving to free operational capacity, but fair-use caps remain in effect.

Rationale:
- Prevents unlimited create+archive loops.

---

### FU-EC-05 — Soft limit warning must not become an entitlement gate

**Scenario:** A module surfaces "soft limit reached" as a blocker or an "upgrade" path.

**Required behavior:**
- Soft-limit signals are warnings/telemetry only.
- Any blocking behavior must use hard-limit denial reasons.

---

### FU-EC-06 — Interaction with entitlements (modules disabled/read-only)

**Scenario:** Inventory is not subscribed (`READ_ONLY`) and the user attempts to create stock items.

**Required behavior:**
- Entitlements (Access Control) deny the write first (`ENTITLEMENT_READ_ONLY`).
- Fair-use denial must not be used as a substitute for entitlement enforcement.

---

### FU-EC-07 — Rate limit vs legitimate burst (setup/import)

**Scenario:** During initial setup, an admin adds many menu items quickly.

**Required behavior:**
- Rate limits should allow reasonable bursts for normal usage.
- If rate limit triggers, the system must degrade gracefully:
  - deny with `FAIRUSE_RATE_LIMITED`,
  - allow retry after the window without data corruption.

---

### FU-EC-08 — Observability requirement

**Scenario:** A tenant repeatedly hits limits.

**Required behavior:**
- Hard limit and rate limit denials must be audit/telemetry observable (platform-level event).
- This is not a user-audit log of operational actions; it is system health visibility.
