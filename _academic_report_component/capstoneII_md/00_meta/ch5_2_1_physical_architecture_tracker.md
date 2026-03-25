# Chapter 5.2.1 Physical Architecture Tracker

Purpose: track the frontend/backend facts still needed to finalize `5.2.1 Physical Architecture` in `_academic_report_component/capstoneII_md/05_detail_concepts/5.md`.

## Already Confirmed

- Modula has a real staging deployment on the internet.
- Frontend is implemented as a Flutter web application.
- Backend is implemented with Node.js and TypeScript.
- The system is designed for offline-first behavior with local persistence, queued offline operations, and synchronization.
- The authoritative database is PostgreSQL.
- Object storage uses Cloudflare R2.
- The backend exposes sync endpoints and uses reliability mechanisms such as duplicate-safe execution and outbox-based post-commit processing.

## Open Questions

| ID | Owner | Needed For | Question | Status | Answer |
|---|---|---|---|---|---|
| CH5-PHY-FE-01 | Frontend | Frontend Delivery | What is the actual frontend hosting provider for staging, and what is the public staging URL? | ANSWERED | Netlify; public URL: `modulapos.com` |
| CH5-PHY-FE-02 | Frontend | Client Layer | In the report, should local persistence be described directly as IndexedDB, or as browser local persistence backed by IndexedDB? | ANSWERED | At the architecture level, local persistence should be described as Drift-based local relational persistence on SQLite. For the web deployment, that SQLite database is persisted through an IndexedDB-backed virtual file system. Offline-first is not yet fully enabled in the current frontend build, so this should be framed as the intended local persistence architecture rather than a fully active staging capability. |
| CH5-PHY-FE-03 | Frontend | Communication Model | Does the staging frontend consume the SSE notification/sync-trigger endpoint directly? | ANSWERED | Yes, but only for operational notifications. The staging frontend directly consumes the SSE notification stream. Synchronization is not driven by a dedicated SSE sync-trigger endpoint; sync is initiated by frontend lifecycle logic such as hydration, tenant switch, branch switch, and reconnect handling, which then calls the normal HTTP sync endpoints. Payment verification status is also checked by polling backend endpoints from the frontend. |
| CH5-PHY-FE-04 | Frontend | Client Layer | Are there other runtime delivery components worth naming in the physical architecture, such as service workers or PWA caching? | ANSWERED | Service workers or PWA caching should not be elevated as primary named runtime components. More faithful runtime elements are: Flutter web client, Drift/SQLite local persistence, SQLite WebAssembly runtime, IndexedDB-backed browser file system for SQLite persistence, Drift web worker, HTTP API integration, and the SSE operational-notification client. |
| CH5-PHY-BE-01 | Backend | Current Staging Deployment | Confirm the backend hosting provider and the staging API/base URL. | ANSWERED | Render; base URL: `https://modulabackend.onrender.com` |
| CH5-PHY-BE-02 | Backend | Database Layer | Confirm the report wording for the authoritative database deployment (for example, Supabase PostgreSQL). | ANSWERED | Database hosting provider: Supabase. |
| CH5-PHY-BE-03 | Backend | Supporting Services | Which OTP provider is currently used in staging? | ANSWERED | Twilio is the selected OTP provider. In staging, a fixed-OTP fallback is currently used because some Cambodian carriers, including Smart Axiata, block SMS from long-code senders. This blocker would require legal business/startup registration to resolve properly. |
| CH5-PHY-BE-04 | Backend | Supporting Services | How should payment verification be described physically in staging? Which provider names can be stated for sale payment and billing payment? | ANSWERED | Cash sale payment: verified directly by cashier receipt of cash, no external provider. KHQR sale payment: verified through Bakong KHQR after customer payment from a banking app. Bakong confirmation is obtained by backend polling/reconciliation rather than provider webhook support. Billing/branch-activation payment in staging should not be described as using a live external gateway yet; backend currently uses a stub/simulated verifier for billing confirmation. |
| CH5-PHY-BE-05 | Backend | Backend Application Layer | Is the outbox dispatcher/background job runner active in staging, and should it be shown in the physical architecture diagram? | ANSWERED | Yes. It is active in staging and starts inside the same backend service process by default rather than as a separate deployed worker. It should appear in the physical architecture diagram as an in-process background runner inside the backend service box. This in-process runner is also the right place to describe Bakong payment-confirmation polling/reconciliation where applicable. |
| CH5-PHY-BE-06 | Backend | Communication Model | Is SSE active in staging as a sync trigger, and should it appear explicitly in the physical architecture diagram? | ANSWERED | Yes, but with narrower scope. SSE is active in staging as the operational-notification stream (`GET /v0/notifications/stream`), not as the main global synchronization mechanism. If shown in the physical architecture diagram, it should be labeled specifically as a backend-to-client SSE notification channel. |
| CH5-PHY-BE-07 | Backend | Logical Architecture Cleanup | Does a real webhook gateway component still exist in current architecture, or should that concept be removed from Chapter 5 entirely? | ANSWERED | A separate webhook gateway component should not be shown as an active standalone component. What currently exists is a direct webhook-ingestion route inside the backend service, specifically the KHQR provider webhook endpoint (`POST /v0/payments/khqr/webhooks/provider`). |
| CH5-PHY-BE-08 | Backend | Supporting Services | Is a CDN currently in use for object-storage or frontend delivery, and if so which provider should be named? | ANSWERED | CDN capability is mostly provider-managed rather than manually built by the team. Netlify already provides edge delivery for frontend static assets. Cloudflare R2 is the confirmed provider for media/object storage. In current staging, upload flow is `client -> backend -> R2`, while media reads are primarily `client -> R2 public URL`. The backend image-proxy route exists as a fallback path, not the main staging read path. |

## Usage Notes

- `CH5-PHY-FE-01`, `CH5-PHY-BE-01`, `CH5-PHY-BE-02`, `CH5-PHY-BE-03`, `CH5-PHY-BE-04`, and `CH5-PHY-BE-08` are the most important for polishing `5.2.1`.
- `CH5-PHY-BE-05`, `CH5-PHY-BE-06`, and `CH5-PHY-BE-07` also affect `5.2.2 Logical Architecture`.
- If a question is answered verbally in chat or by a teammate, record the answer here immediately so the detail is not lost before final rewriting.
