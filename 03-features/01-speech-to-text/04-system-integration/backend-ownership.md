# 04. System Integration — Backend Ownership

## Purpose

Define backend workstream responsibility and implementation boundary. This is a work breakdown, not a statement that tasks are complete or that exact APIs/schemas have been approved.

## Scope and prerequisites

ASP.NET Core meeting integration, authenticated audio gateway, internal AI transport, transcript events/storage/history. The Python inference engine belongs to AI ownership; backend owns its service integration. Requires architecture, requirements and versioned contracts.

## Work items and dependencies

| ID | Work item | Dependencies | Acceptance evidence |
|---|---|---|---|
| BE-STT-001 | Meeting/participant/transcript persistence model and EF Core migration. | Existing identity/meeting schema; approved data model. | Migration applies/rolls back as supported; constraints, indexes and retention behavior reviewed. |
| BE-STT-002 | Meeting lifecycle and membership endpoints/integration. | BE-STT-001 or existing meeting service. | Authorization and membership state tests. |
| BE-STT-003 | Authenticated audio WebSocket gateway and per-stream bounded channel. | BE-STT-002, streaming contract. | Invalid auth/member rejected; concurrent streams isolated; limits and cancellation verified. |
| BE-STT-005 | Internal audio forwarder and AI response client. | BE-STT-003; AI service contract/readiness. | Contract, timeout, retry, backpressure and correlation tests. |
| BE-STT-006 | Authorized real-time transcript delivery (SignalR proposed). | BE-STT-005; participant authorization. | Correct meeting audience, ordering, partial replacement and final idempotency verified. |
| BE-STT-007 | Final transcript persistence worker and paginated history API. | BE-STT-001, BE-STT-006. | Durable final records, authorized retrieval, ordering and pagination tests. |

Original planning estimates (hours, O/M/P) are retained for scheduling context: BE-STT-001 6/10/16; 002 8/12/20; 003 12/18/28; 005 10/16/24; 006 8/12/18; 007 6/10/14. These are unvalidated estimates, not commitments. The previous `BE-STT-004` ID described the Python FastAPI/VAD inference service and is assigned to the AI workstream, not backend implementation ownership.

## Technical approach

- Derive identity from authenticated claims and validated meeting state; never accept client-supplied speaker identity.
- Keep audio buffering bounded per stream and release on every close, cancellation and error path.
- Use internal service credentials and correlation IDs; do not forward external JWT secrets unnecessarily.
- Separate partial broadcast from final persistence; make final writes idempotent and ordered using an agreed sequence/time model.
- Enforce authorization on connection admission, hub subscription, broadcast targeting and history query.
- Define retry/circuit behavior with idempotency and backpressure; retries must not duplicate persisted transcript segments.

## Tools and technologies

Project drafts reference ASP.NET Core, EF Core, SQL Server, `System.Threading.Channels`, SignalR and Polly. These are candidates consistent with the stated project context; verify existing repository/platform conventions and versions. Redis backplane and specific endpoint routes remain proposals.

## Deliverables

Approved entity/API/event schemas; migrations; gateway and AI client; broadcast and persistence services; auth/security tests; operational metrics and failure handling.

## Acceptance criteria

All interfaces match the published contract; no unauthorized cross-room flow; bounded memory and cleanup under disconnect; finalized segments persist once with offsets and stable participant identity; histories are paginated and authorized; failures surface actionable status without exposing secrets.

## Dependencies and open questions

Depends on identity/meeting services, AI contract, storage policy and SignalR/platform decisions. Decide transcript schema and timebase, message ordering, durable queue needs, retry semantics, retention/deletion, API routes and deployment scaling.
