# Phase 6: Backend Development Tasks (WBS)

## 1. Workstream Overview

This document specifies the complete engineering Work Breakdown Structure (WBS) for the Backend workstream, spanning both the **ASP.NET Core Web API Core** (`backend/`) and the **Python FastAPI AI Engine** (`ai-services/`).

All tasks follow the project's standardized task schema and three-point PERT estimation model ($E = \frac{O + 4M + P}{6}$).

---

## 2. Tasks Catalog: Backend & AI Services

### `BE-STT-001`: Meeting & Transcript Entity Framework Core Mappings
- **Task ID**: `BE-STT-001`
- **Task Name**: Implement Meeting, Participant, and Transcript EF Core Schema
- **Description**: Add `Meeting`, `MeetingParticipant`, and `TranscriptSegment` domain entities, configure EF Core fluent mappings, apply database indexes, and generate the migration script.
- **Priority**: Critical
- **Responsible Department**: Backend Engineering (.NET)
- **Dependencies**: None (builds on existing `ApplicationDbContext`)
- **Implementation Steps**:
  1. Create domain entities in `AiraMeet.Domain/Entities/`.
  2. Implement `IEntityTypeConfiguration` classes in `AiraMeet.Infrastructure/Data/Configurations/`.
  3. Register `DbSet<Meeting>`, `DbSet<MeetingParticipant>`, `DbSet<TranscriptSegment>` in `ApplicationDbContext`.
  4. Create and verify EF Core migration: `dotnet ef migrations add AddMeetingAndTranscripts`.
- **Time Estimates**:
  - Optimistic ($O$): 6h
  - Most Likely ($M$): 10h
  - Pessimistic ($P$): 16h
  - Expected ($E$): $\frac{6 + 4(10) + 16}{6} = 10.33\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Database migration applies cleanly to SQL Server; relational integrity and cascading deletes function as specified.
- **Deliverable**: C# entity classes and EF Core migration files.
- **Status**: `To Do`

---

### `BE-STT-002`: Meeting Lifecycle Management APIs
- **Task ID**: `BE-STT-002`
- **Task Name**: Implement Meeting Creation, Join, and Leave REST Endpoints
- **Description**: Build meeting orchestration endpoints enabling authenticated users to create meetings, generate unique join codes, join active meetings, and leave meetings.
- **Priority**: High
- **Responsible Department**: Backend Engineering (.NET)
- **Dependencies**: `BE-STT-001`
- **Implementation Steps**:
  1. Define DTOs (`CreateMeetingRequest`, `MeetingResponse`, `JoinMeetingResponse`).
  2. Implement `IMeetingService` in `AiraMeet.Application`.
  3. Implement `MeetingsController` in `AiraMeet.Api` with `[Authorize]` policy.
  4. Ensure meeting host permissions and participant tracking logic.
- **Time Estimates**:
  - Optimistic ($O$): 8h
  - Most Likely ($M$): 12h
  - Pessimistic ($P$): 20h
  - Expected ($E$): $\frac{8 + 4(12) + 20}{6} = 12.67\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Authenticated user can create a meeting, join via 10-character code, and list meeting participants.
- **Deliverable**: `MeetingsController.cs` and `MeetingService.cs` with unit tests.
- **Status**: `To Do`

---

### `BE-STT-003`: Authenticated Audio Streaming WebSocket Gateway
- **Task ID**: `BE-STT-003`
- **Task Name**: Implement Inbound Audio Streaming WebSocket Gateway
- **Description**: Build an ASP.NET Core WebSocket middleware/handler at `/ws/meetings/{id}/audio-stream` to receive raw 16 kHz PCM frames and derive participant identity from JWT claims.
- **Priority**: Critical
- **Responsible Department**: Backend Engineering (.NET)
- **Dependencies**: `BE-STT-001`, `BE-STT-002`
- **Implementation Steps**:
  1. Implement WebSocket upgrade middleware with Bearer JWT validation.
  2. Extract `UserId` from authenticated token and bind to internal connection channel.
  3. Reject unauthenticated handshakes with `401 Unauthorized`.
  4. Buffer incoming binary frames in thread-safe memory channels (`System.Threading.Channels.Channel<byte[]>`).
- **Time Estimates**:
  - Optimistic ($O$): 12h
  - Most Likely ($M$): 18h
  - Pessimistic ($P$): 28h
  - Expected ($E$): $\frac{12 + 4(18) + 28}{6} = 18.67\text{h}$
  - Risk Buffer: 3h
- **Acceptance Criteria**: Server ingests binary frames from multiple concurrent connections, maintaining strict identity isolation with zero memory leaks.
- **Deliverable**: `AudioStreamGatewayMiddleware.cs` and `AudioChannelManager.cs`.
- **Status**: `To Do`

---

### `BE-STT-004`: Python FastAPI Inference Microservice
- **Task ID**: `BE-STT-004`
- **Task Name**: Implement FastAPI Streaming STT Engine with VAD
- **Description**: Construct the Python FastAPI AI service with Silero Voice Activity Detection, 1.5-second sliding window buffer, and CTranslate2 model execution.
- **Priority**: Critical
- **Responsible Department**: AI & Machine Learning
- **Dependencies**: `AI-STT-003` (Fine-tuned model checkpoint)
- **Implementation Steps**:
  1. Initialize FastAPI project with async WebSocket / gRPC endpoint in `ai-services/`.
  2. Integrate `silero-vad` for silence classification.
  3. Implement circular buffer with 300ms overlap.
  4. Load quantized CTranslate2 speech model on GPU.
  5. Implement boundary deduplication algorithm.
- **Time Estimates**:
  - Optimistic ($O$): 14h
  - Most Likely ($M$): 20h
  - Pessimistic ($P$): 32h
  - Expected ($E$): $\frac{14 + 4(20) + 32}{6} = 21.00\text{h}$
  - Risk Buffer: 4h
- **Acceptance Criteria**: Service transcribes 1.5s audio chunk in $\le 250\text{ms}$ on target GPU, emitting partial and final transcripts.
- **Deliverable**: `ai-services/app/streaming_engine.py` and API endpoints.
- **Status**: `To Do`

---

### `BE-STT-005`: Inter-Service Communication & Forwarding Client
- **Task ID**: `BE-STT-005`
- **Task Name**: Integrate Backend Audio Forwarder to AI Service
- **Description**: Build high-performance internal transport (gRPC or persistent WebSocket) connecting ASP.NET Core audio channels to the FastAPI AI inference engine.
- **Priority**: High
- **Responsible Department**: Backend Engineering (.NET)
- **Dependencies**: `BE-STT-003`, `BE-STT-004`
- **Implementation Steps**:
  1. Define protobuf schema or internal streaming client.
  2. Implement async worker in ASP.NET Core reading from audio channels and streaming to FastAPI.
  3. Receive partial and final transcription packets from FastAPI.
  4. Implement circuit breaker and retry policy via Polly.
- **Time Estimates**:
  - Optimistic ($O$): 10h
  - Most Likely ($M$): 16h
  - Pessimistic ($P$): 24h
  - Expected ($E$): $\frac{10 + 4(16) + 24}{6} = 16.33\text{h}$
  - Risk Buffer: 3h
- **Acceptance Criteria**: Backend successfully streams audio frames to AI service and receives transcribed text with latency under 100ms overhead.
- **Deliverable**: `SttInferenceClient.cs` and background streaming worker.
- **Status**: `To Do`

---

### `BE-STT-006`: Real-Time Transcript Broadcast Hub (SignalR)
- **Task ID**: `BE-STT-006`
- **Task Name**: Implement Live Transcript Broadcast SignalR Hub
- **Description**: Build ASP.NET Core SignalR hub `/hubs/meeting-transcripts` to multicast live transcript events (partial and final) to all room participants.
- **Priority**: High
- **Responsible Department**: Backend Engineering (.NET)
- **Dependencies**: `BE-STT-005`
- **Implementation Steps**:
  1. Create `MeetingTranscriptHub : Hub<ITranscriptClient>`.
  2. Group participants by `meetingId`.
  3. Dispatch `ReceiveTranscriptSegment` containing speaker name, avatar, text, and `is_final` flag.
  4. Configure Redis backplane for horizontal scale.
- **Time Estimates**:
  - Optimistic ($O$): 8h
  - Most Likely ($M$): 12h
  - Pessimistic ($P$): 18h
  - Expected ($E$): $\frac{8 + 4(12) + 18}{6} = 12.33\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Transcribed speech emitted by the AI service appears in the SignalR group within 50ms of receipt.
- **Deliverable**: `MeetingTranscriptHub.cs` and Redis backplane configuration.
- **Status**: `To Do`

---

### `BE-STT-007`: Transcript Persistence Worker & Query APIs
- **Task ID**: `BE-STT-007`
- **Task Name**: Implement Transcript Persistence Worker and History REST API
- **Description**: Persist final transcript chunks to SQL Server and implement paginated retrieval endpoint `GET /api/meetings/{id}/transcripts`.
- **Priority**: Medium
- **Responsible Department**: Backend Engineering (.NET)
- **Dependencies**: `BE-STT-001`, `BE-STT-006`
- **Implementation Steps**:
  1. Add background queue worker to write `TranscriptSegment` entities asynchronously without blocking real-time broadcast.
  2. Implement paginated query with speaker metadata in `MeetingController`.
  3. Add caching for completed meetings.
- **Time Estimates**:
  - Optimistic ($O$): 6h
  - Most Likely ($M$): 10h
  - Pessimistic ($P$): 14h
  - Expected ($E$): $\frac{6 + 4(10) + 14}{6} = 10.00\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Final transcripts stored with millisecond offsets; GET endpoint returns paginated transcript history.
- **Deliverable**: `TranscriptPersistenceWorker.cs` and query endpoints.
- **Status**: `To Do`
