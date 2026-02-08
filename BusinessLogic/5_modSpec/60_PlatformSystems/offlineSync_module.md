# Sync & Offline Module — Core Module (Capstone 1)

**Version:** 1.0  
**Status:** Defined (Capstone 1)  
**Module Type:** Core Module  
**Depends on:** Authentication & Authorization (Core), Tenant & Branch Context (Core), Audit Logging (Core)  
**Related Modules:** Sale, Cash Session, Attendance, Menu, Inventory

---


## 1. Purpose

The Sync & Offline module ensures that **Modula remains operational even with unstable or unavailable internet**, especially in the cashier environment.  
It provides:

- Local storage for sales, cash session actions, and attendance logs  
- A durable queue for pending operations  
- Automatic synchronization when connectivity returns  
- Idempotent delivery of operations to the backend  
- A unified offline/online status for UI modules  

This module guarantees that:

- Cashiers can continue selling during outages  
- Cash sessions can open/close offline  
- Attendance (tied to sessions) still functions offline  
- No double-submissions occur when resyncing  
- Other modules do not need to implement offline logic themselves  

---

## 2. Scope & Boundaries

### This module covers:

- Local storage engine (IndexedDB or device local DB)
- Local caching of:
  - Menu items  
  - Modifier groups  
  - Policy snapshot  
  - User identity + tenant + branch  
- Operation queue system:
  - Creating queued operations for offline actions  
  - Retrying sync when connection is restored  
  - Ensuring event ordering  
- Idempotency:
  - Unique client operation IDs  
  - Backend-safe retry guarantees  
- Connectivity detection:
  - Online → offline transitions  
  - UI-facing connectivity state  

### This module does NOT cover:

- Inventory restocking (online-only in Capstone 1)
- Menu editing (online-only)  
- Policy editing (online-only)  
- Staff management flows  
- Sync conflict UI for admins (future)  
- Continuous background analytics syncing (future)  

### Dependencies:

- Auth module  
- Sale module (uses sync engine to finalize sales)  
- Cash Session module  
- Attendance module  
- Policy module  
- Menu module  
- Tenant & Branch context  

---

## 3. Use Cases (with actor + permission)

---

### UC-1: Cashier finalizes a sale while offline

**Actor:** Cashier  
**Permissions:** Cashier role with active tenant & branch  

**Preconditions:**
- Device has local copies of menu, policies, and user info  
- Sale finalize normally requires network but must still function offline  

**Main Flow:**
1. Cashier selects items and presses "Checkout".  
2. System detects offline state (or server request fails).  
3. Sync module:
   - Creates a queued operation `{ type: SALE_FINALIZED, payload, client_op_id }`
   - Generates a temporary local sale record with a local-only ID  
4. UI shows:
   - “Sale processed offline; syncing when connection returns.”  
5. When connection returns, sync module replays the operation to backend.

**Acceptance:**
- Cashier can continue working without interruption.  
- Backend creates the official sale using client_op_id.  

---

### UC-2: Cashier opens a cash session offline

**Actor:** Cashier  
**Permissions:** Cashier/Manager depending on branch scope  

**Preconditions:**
- No existing open session on device  
- Cash Session policy allows session operations  

**Main Flow:**
1. Cashier selects “Start Session”.  
2. Offline state is detected.  
3. Sync module stores:
   - Session open event with opening float and `client_op_id`  
4. UI shows the session as active, even though not yet synced.  
5. When online, sync module sends the open event to server.  

**Acceptance:**  
- Session UI behaves as if online.  
- Server eventually creates the session with correct timestamps.

---

### UC-3: Cash Session movements (Paid In, Paid Out, Close) offline

**Actor:** Cashier / Manager  

**Preconditions:**
- An offline or online session exists  

**Main Flow:**
1. Cashier performs Paid In, Paid Out, or session close.  
2. Sync module logs each movement as a queued operation.  
3. UI reflects updated session state instantly.  
4. Sync sends them when back online in strict order:
   - Open → Movements → Close  

**Acceptance:**  
- No double posting  
- Movements preserved even if device crashes  

---

### UC-4: Attendance check-in via session start (offline)

**Actor:** Cashier  

**Preconditions:**
- Attendance policy: cash-session-as-attendance = ON  

**Main Flow:**
1. Cashier starts a cash session offline.  
2. Sync module creates:
   - `ATTENDANCE_CHECK_IN` event  
3. Attendance log remains local until synced.

**Acceptance:**  
- Staff are considered checked-in even offline  
- Attendance module receives correct timestamps when synced  

---

### UC-5: Sync engine retries pending operations when online

**Actor:** System module  

**Preconditions:**
- Device reconnects to the internet  
- Queue contains pending operations  

**Main Flow:**
1. Connectivity monitor detects online state.  
2. Sync engine processes queue in FIFO order.  
3. For each operation:
   - Calls backend with `client_op_id`
   - Backend ensures idempotency  
4. If success: mark as `synced`  
5. If permanent failure: mark as `failed` with reason  

**Acceptance:**  
- No duplicated sales or sessions  
- Queue drains automatically  

---

### UC-6: UI displays offline/online status and pending sync count

**Actor:** Cashier  

**Main Flow:**
- Sync module exposes:
  - Connection status  
  - Pending operations count  
- UI shows:
  - “Online”
  - “Offline • 3 operations pending”
  - “Syncing…”

**Acceptance:**  
- User receives clear feedback on system state  
- No confusing silent failures  

---

## 4. Functional Requirements (FR)

### Local Data Storage

- **FR-1:** Menu, policies, and user context must be cached offline.  
- **FR-2:** Cached data must persist across app/browser restarts.  
- **FR-3:** Local data must be tenant- and branch-specific.

### Operation Queue

- **FR-4:** All offline operations must be stored in a durable queue.  
- **FR-5:** Each queued op must have:
  - `client_op_id (UUID)`
  - `type`
  - `payload`
  - `timestamp`
  - `status (pending/syncing/synced/failed)`
- **FR-6:** Operations must preserve order (FIFO).  
- **FR-7:** Deleting or reordering operations is forbidden unless system-approved.

### Idempotency

- **FR-8:** All queued operations must be idempotent via client_op_id.  
- **FR-9:** Backend must either:
  - Create the record  
  - Or return the already-created record for the same client_op_id  

### Connectivity Monitoring

- **FR-10:** Module must detect:
  - offline → online  
  - online → offline  
- **FR-11:** Events should be exposed to UI for display.  

### Sync Logic

- **FR-12:** Sync loop triggered on reconnect and periodically (e.g., every X seconds).  
- **FR-13:** Sync must stop if:
  - User logs out  
  - Tenant switches  
  - Session closes  

### Error Handling

- **FR-14:** Retry transient errors.  
- **FR-15:** If permanent failure, mark operation as failed and expose to UI.

### Interaction with Feature Modules

- **FR-16:** Sale, Cash Session, Attendance must call `queueOperation()` instead of writing directly when offline.  
- **FR-17:** Feature modules must remain unaware of local DB internals.  
- **FR-18:** Policies and menu must be loaded from local cache first, server second.

---

## 5. Acceptance Criteria (AC)

- **AC-1:** Cashier can complete sales fully offline.  
- **AC-2:** Cash session open/close works offline.  
- **AC-3:** Attendance check-in works offline when tied to cash session start.  
- **AC-4:** Local data persists after app/browser restart.  
- **AC-5:** Upon reconnection, all pending operations sync without duplication.  
- **AC-6:** UI shows correct offline/online status at all times.  
- **AC-7:** UX never freezes or blocks cashier actions during outage.  
- **AC-8:** Backend receives consistent, idempotent events even after multiple retries.  
- **AC-9:** If sync fails permanently, UI clearly informs the user or admin.

---

## 6. Out of Scope (Capstone 1)

- Conflict UI (e.g., sale references menu item deleted on server)  
- Offline restocking  
- Offline menu editing  
- Offline policy editing  
- Advanced sync scheduling  
- Multi-device sync reconciliation for the same cashier  
- Transaction snapshots for auditing

# Sync & Offline Module — Core Module (Capstone I)

**Version:** 1.2  
**Status:** Patched (Cash sessions required for sales; frozen-branch enforcement)  
**Module Type:** Core Module  
**Depends on:** Authentication & Authorization (Core), Tenant & Branch Context (Core), Audit Logging (Core)  
**Related Modules:** Sale, Cash Session, Attendance, Menu, Inventory

---

## 1. Purpose

The Sync & Offline module ensures that **Modula remains operational with unstable or unavailable connectivity**, while **preserving backend data integrity**.

It provides:
- Local persistence and caching for offline operation
- A durable client-side operation queue
- Idempotent, ordered delivery to the backend
- Clear UX signals for offline/online state
- Server-authoritative integrity when processing queued operations

This module guarantees that:
- Cashiers can continue working during outages
- Finalized sales remain **exactly-once** on the backend
- Inventory and cash mutations remain **atomic and concurrency-safe**
- Offline behavior does **not** weaken integrity (exactly-once, transactional guarantees)
- Writes targeting **frozen branches** are rejected deterministically

---

## 2. Scope & Boundaries

### This module covers
- Local storage engine (IndexedDB / device local DB)
- Local caching of:
  - Menu items
  - Policy snapshot
  - User identity + tenant + branch context
- Operation queue:
  - Enqueueing offline operations
  - Retrying on reconnect
  - Preserving FIFO order per device
- Idempotency:
  - Client-generated operation IDs
  - Backend-safe retry semantics
- Connectivity monitoring and UI exposure

### This module does NOT cover
- Offline menu editing
- Offline inventory restocking
- Offline policy editing
- Admin conflict-resolution UI
- Background analytics syncing

---

## 3. Core Integrity Rules (Patched)

### 3.1 Server-Authoritative Ordering (Locked)

Client-side timestamps are **not authoritative** for integrity.

Rules:
- Each queued operation includes a `client_op_id` and a client timestamp (for audit only).
- The backend assigns authoritative ordering at processing time.
- Integrity (stock, cash, sale finalization) depends on **backend transaction order**, not client order.

Implication:
- Multiple devices may enqueue operations concurrently.
- The backend processes operations safely using database transactions and constraints.

---

### 3.2 Idempotency & Exactly-Once Semantics (Locked)

All queued write operations **MUST** be idempotent.

Rules:
- Every queued operation includes a globally unique `client_op_id`.
- Backend behavior:
  - If `client_op_id` is new → process operation.
  - If `client_op_id` already processed → return existing result.
- Retries (network drops, offline replay, double submit) **must not** create duplicates.

---

### 3.3 Cash Session Requirement (Patched)

**Product rule (Capstone 1):** cash sessions are required for cart/sales.

Offline implications:
- Offline queues must include cash-session actions (open/close) and preserve operation order.
- A cash sale finalized offline must only be applied if an OPEN cash session exists for the user at that branch (or the operation is rejected deterministically).

---

### 3.4 Frozen Branch Enforcement (New, Locked)

Branch state is authoritative on the backend.

Rules:
- Queued operations targeting a **frozen branch** MUST be rejected by the backend.
- Rejection behavior:
  - Deterministic error code (e.g., `BRANCH_FROZEN`)
  - No retries for the same operation
- Client behavior:
  - Mark operation as `failed`
  - Surface clear reason to user
  - Prevent retry storms

Acceptance:
- “Queued writes for frozen branches fail deterministically with clear error codes.”

---

## 4. Use Cases (Selected)

### UC-1: Finalize Sale While Offline

**Actor:** Cashier  
**Flow:**
1. Cashier finalizes sale offline.
2. Sync module enqueues:
   - `{ type: SALE_FINALIZED, payload, client_op_id }`
3. UI confirms offline completion.
4. On reconnect, backend processes finalize atomically and idempotently.

**Guarantee:** Exactly-once sale creation; no duplicate inventory or cash effects.

---

### UC-2: Sync Loop on Reconnect

**Actor:** System  
**Flow:**
1. Connectivity restored.
2. Sync processes queue FIFO.
3. Backend validates:
   - Branch state (active vs frozen)
   - Idempotency
4. Success → mark `synced`
5. Permanent failure → mark `failed` with reason

---

## 5. Functional Requirements (FR)

### Queue & Ordering
- **FR-1:** All offline writes must be queued durably.
- **FR-2:** Each operation includes `client_op_id`, payload, timestamp, branch_id.
- **FR-3:** Backend processing order is authoritative.

### Idempotency
- **FR-4:** Backend must enforce exactly-once semantics using `client_op_id`.

### Branch State
- **FR-5:** Backend must reject operations for frozen branches.
- **FR-6:** Client must not retry frozen-branch failures.

### UX
- **FR-7:** UI must show offline/online state and pending count.
- **FR-8:** UI must surface clear failure reasons for rejected sync operations.

---

## 6. Acceptance Criteria (AC)

- **AC-1:** Offline sales sync exactly once.
- **AC-2:** Concurrent finalize operations remain consistent.
- **AC-3:** Cash-session-disabled tenants remain integrity-safe.
- **AC-4:** Frozen branches reject queued writes deterministically.
- **AC-5:** Users receive clear feedback for sync failures.

---

## 7. Audit Events Emitted

- `SYNC_QUEUE_ENQUEUED`
- `SYNC_QUEUE_DEQUEUED`
- `SYNC_REJECTED_BRANCH_FROZEN`
- `SYNC_RETRY_SCHEDULED`
- `SYNC_OPERATION_APPLIED`

---

## 8. Out of Scope (Capstone I)

- Cross-device conflict resolution UI
- Offline admin operations
- Advanced sync scheduling
- Post-close refund reconciliation

---

## 7. Developer Notes

- Sync module should be platform-agnostic; feature modules must not embed offline logic.  
- IndexedDB (web) or SQLite/local DB (mobile) should be abstracted behind a simple interface:
  - `queueOperation()`
  - `getPendingOperations()`
  - `runSyncLoop()`
- The simpler the interface, the easier it is to maintain.  
- Sync should avoid blocking the UI thread.  
- Ensure `tenant_id` and `branch_id` are always stored with offline events.

## Audit Events Emitted

The following events MUST be written to the Audit Log when triggered (with tenant_id, branch_id where applicable, actor_id, and relevant entity IDs):

- `SYNC_QUEUE_ENQUEUED`
- `SYNC_QUEUE_DEQUEUED`
- `SYNC_CONFLICT_DETECTED`
- `SYNC_RETRY_SCHEDULED`
