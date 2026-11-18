# System Architecture Specification

## 1. High-Level Architectural Vision

AiraMeet is designed as an asynchronous, event-driven, distributed conferencing and AI processing platform. The architecture separates latency-critical interactive media paths (WebRTC audio/video) from compute-intensive batch/streaming deep learning inference pipelines (FastAPI Speech-to-Text).

```mermaid
flowchart TB
    subgraph ClientZone["Client Zone (Browser / Mobile)"]
        UI["Angular Single Page Application"]
        Worklet["Web Audio API / AudioWorklet<br/>(16 kHz Mono PCM Capture)"]
        UI --- Worklet
    end

    subgraph IngressZone["Ingress & Edge Routing"]
        Ingress["Reverse Proxy / API Gateway<br/>(TLS Termination, Load Balancing)"]
    end

    subgraph CoreBackend["Application Core (ASP.NET Core)"]
        AuthModule["Identity & Auth Module<br/>(JWT, Tokens, ApplicationUser)"]
        MeetingModule["Meeting Orchestration<br/>(Rooms, Participants, State)"]
        AudioHub["Audio Streaming Hub<br/>(SignalR / WebSocket Gateway)"]
        TranscriptHub["Transcript Dispatcher Hub<br/>(Live Broadcast Service)"]
        PersistenceMgr["Data Persistence Worker<br/>(EF Core Unit of Work)"]
    end

    subgraph DataStorage["Persistence & State Tier"]
        SqlServer[("SQL Server<br/>Users, Meetings, Transcripts")]
        RedisStore[("Redis Cluster<br/>Session State, Pub/Sub Bus")]
    end

    subgraph AIServiceTier["AI Intelligence Tier (Python FastAPI)"]
        AudioConsumer["Stream Ingestion Gateway<br/>(Buffer, Normalizer, Resampler)"]
        VADEngine["Voice Activity Detector (VAD)<br/>(Silero / WebRTC VAD)"]
        InferenceWorker["Deep Learning Inference Engine<br/>(Whisper / Wav2Vec2 / NeMo)"]
        TextPostProcessor["Transcript Normalizer<br/>(Deduplication & Punctuation)"]
    end

    UI <-->|HTTPS / REST| Ingress
    Worklet -->|WSS Binary PCM Frames| Ingress
    Ingress <-->|HTTPS / WSS| AuthModule
    Ingress <-->|HTTPS / WSS| AudioHub
    Ingress <-->|WSS Events| TranscriptHub

    AudioHub <--> RedisStore
    TranscriptHub <--> RedisStore
    AuthModule <--> SqlServer
    MeetingModule <--> SqlServer
    PersistenceMgr <--> SqlServer

    AudioHub -->|Internal WSS / gRPC| AudioConsumer
    AudioConsumer --> VADEngine
    VADEngine -->|Speech Chunks| InferenceWorker
    InferenceWorker --> TextPostProcessor
    TextPostProcessor -->|Partial & Final Transcripts| AudioConsumer
    AudioConsumer -->|Transcript Payloads| TranscriptHub
```

---

## 2. Tier-by-Tier Architectural Breakdown

### 2.1 Client Tier (Angular 17+)
- **Presentation Components**: Render authenticated routes (`/dashboard`, `/meeting/:id`, `/transcripts/:id`).
- **Reactive State Layer**: Uses Angular Signals and RxJS Subject streams for managing participant connection states, mute/unmute indicators, and live transcript scrolling.
- **Audio Worklet Subsystem**: Bypasses the UI main thread using an `AudioWorkletProcessor` to tap the microphone `MediaStream`. Captures raw 32-bit Float samples, downsamples to 16 kHz 16-bit signed integer linear PCM, and streams chunks every 250ms over a dedicated WebSocket connection.

### 2.2 Application Core Tier (ASP.NET Core Web API)
- **Security Boundary**: Validates Bearer JWT tokens on every REST endpoint and upgrades WebSockets only after validating token claims.
- **Connection Registry**: Maintains active meeting room topologies in Redis memory. When a client opens an audio streaming socket, the server binds the socket connection directly to the authenticated `ApplicationUser.Id` and `MeetingRoom.Id`.
- **Media Ingestion & Forwarding**: Buffers participant audio frames into rolling queues and forwards them to the AI Service Tier via gRPC or persistent WebSocket tunnels.
- **Transcript Broadcast Hub**: Consumes recognized text tokens from the AI tier, validates permissions, persists finalized segments into SQL Server, and multicasts live updates to all meeting room participants.

### 2.3 AI Intelligence Tier (Python FastAPI & PyTorch)
- **Decoupled Compute**: The AI service operates independently of the web application. It can scale horizontally based on GPU availability and meeting concurrency.
- **Voice Activity Detection (VAD)**: Inspects incoming audio frames to classify frames as speech vs. non-speech (silence, background noise). Non-speech frames are discarded before hitting heavy neural network layers.
- **Streaming Audio Buffer**: Implements a circular sliding window (1.0 to 2.0-second context window with 200–400ms overlap).
- **Inference Engine**: Executes forward passes on selected deep learning models (Whisper, Wav2Vec 2.0, or NeMo Conformer). Emits **Partial Transcripts** (interim hypothesis) and **Final Transcripts** (committed text with boundary timestamps).

### 2.4 Data & Persistence Tier
- **SQL Server**: Stores relational entities, transactional audit trails, user identities, meeting attendance logs, and permanent meeting transcripts.
- **Redis**: Provides sub-millisecond ephemeral state tracking, token blocklists, active meeting participant rosters, and cross-node message distribution for multi-instance ASP.NET Core clusters.

---

## 3. End-to-End Real-Time Data Flow

The following sequence diagram and narrative detail the end-to-end data lifecycle from the moment a participant un-mutes and speaks until synchronized, speaker-attributed transcripts are rendered across all room clients and persisted to long-term storage.

```mermaid
sequenceDiagram
    autonumber
    actor Speaker as Meeting Speaker
    participant SpeakerApp as Speaker Angular Client
    participant NetGateway as ASP.NET Core Gateway (API & SignalR)
    participant Redis as Redis State & Backplane
    participant AIService as FastAPI AI Service (GPU Worker)
    participant DB as SQL Server
    participant PeerApps as Peer Angular Clients

    Note over Speaker,PeerApps: Phase 1: Connection & Cryptographic Session Binding
    SpeakerApp->>NetGateway: WSS Handshake /ws/meetings/{id}/audio-stream (Bearer JWT)
    NetGateway->>NetGateway: Validate JWT & Extract UserId claim
    NetGateway->>Redis: Register active session channel (SocketId <-> UserId)
    NetGateway-->>SpeakerApp: 101 Switching Protocols (Stream Ready)

    Note over Speaker,PeerApps: Phase 2: Live Audio Capture & Streaming Loop
    Speaker->>SpeakerApp: Speaks into microphone
    SpeakerApp->>SpeakerApp: AudioWorklet encodes 16 kHz Mono Linear PCM
    SpeakerApp->>NetGateway: Send binary PCM chunk (250ms frame, no client-side ID)
    NetGateway->>NetGateway: Retrieve bound UserId from internal session context
    NetGateway->>AIService: Internal gRPC Stream (AudioChunk + verified UserId)

    Note over Speaker,PeerApps: Phase 3: VAD, Inference & Live Partial Transcript
    AIService->>AIService: Silero VAD detects active speech phonemes
    AIService->>AIService: Buffer into 1.5s sliding window (300ms overlap)
    AIService->>AIService: Deep Learning Inference (CTranslate2 INT8)
    AIService-->>NetGateway: Interim Hypothesis: recognized_text, is_final=false
    NetGateway->>Redis: Publish interim event to meeting room channel
    NetGateway-->>SpeakerApp: SignalR Live Caption: is_final=false (Interim italics)
    NetGateway-->>PeerApps: SignalR Live Caption: is_final=false (Interim italics)

    Note over Speaker,PeerApps: Phase 4: Pause Detection, Deduplication & Segment Finalization
    Speaker->>SpeakerApp: Pauses speaking (>= 500ms silence detected)
    AIService->>AIService: VAD triggers segment finalization
    AIService->>AIService: Longest common substring prefix pruning (Deduplication)
    AIService-->>NetGateway: Final Segment: full text, start/end ms, confidence, is_final=true

    Note over Speaker,PeerApps: Phase 5: Asynchronous Persistence & Multicast
    NetGateway->>DB: Async persist TranscriptSegment (UserId, MeetingId, Offsets, Text)
    NetGateway->>Redis: Publish final segment to meeting room channel
    NetGateway-->>SpeakerApp: SignalR Final Segment: is_final=true (Committed text)
    NetGateway-->>PeerApps: SignalR Final Segment: is_final=true (Committed text)
```

### 3.1 Detailed Step-by-Step Flow Description

1. **Cryptographic Handshake & Session Binding (Steps 1–4)**:
   - The Angular client establishes a secure WebSocket connection at `/ws/meetings/{id}/audio-stream`.
   - The ASP.NET Core server verifies the cryptographically signed JWT in the handshake headers, extracts the verified `UserId`, and stores the association in memory and Redis.
   - Binary audio frames transmitted over this socket contain raw PCM bytes only; the backend derives speaker identity exclusively from this connection context.

2. **Client Audio Ingestion (Steps 5–9)**:
   - The client `AudioWorkletProcessor` captures microphone audio off the main browser thread, downsamples it to 16 kHz mono 16-bit linear PCM, and streams binary chunks every 250 milliseconds.
   - ASP.NET Core attaches the authenticated `UserId` and forwards the chunk via high-speed internal gRPC or persistent WebSocket to the FastAPI AI service.

3. **VAD, Sliding Window & Partial Inference (Steps 10–16)**:
   - Silero Voice Activity Detection filters out non-speech frames and static noise.
   - Speech frames are accumulated into a 1.5-second sliding window with a 300ms overlap.
   - The deep learning model emits partial (interim) recognition hypotheses, which are multicasted via SignalR to all meeting attendees to deliver live captions with $< 3.0$ seconds visible latency.

4. **Speech Pause Detection & Overlap Deduplication (Steps 17–20)**:
   - When the speaker pauses ($\ge 500\text{ms}$ of contiguous silence) or reaches the 15-second utterance limit, the AI engine triggers segment boundary commitment.
   - An algorithmic text deduplication filter prunes overlapping tokens from consecutive windows using longest common substring alignment.

5. **Asynchronous Persistence & Client Commit (Steps 21–24)**:
   - The backend persistence worker writes the `TranscriptSegment` entity to SQL Server without blocking the real-time broadcast.
   - A final transcript payload (`is_final: true`) with the speaker's name, avatar URL, start/end millisecond offsets, and text is broadcast via SignalR to all room participants, locking the text into the transcript timeline.

---

## 4. Security & Isolation Safeguards

1. **Authentication Binding**: Audio frames never contain unverified metadata fields like `author_id`. Identity is permanently resolved from the initial cryptographically signed JWT presented during the WebSocket handshake.
2. **Network Isolation**: The AI Service Tier does not expose public endpoints. It resides entirely within a secure internal Docker network or Kubernetes pod overlay, reachable only by authenticated ASP.NET Core backend instances.
3. **Data at Rest & Transit**: All external traffic is TLS 1.3 encrypted (HTTPS/WSS). Audio data buffered in memory is overwritten immediately upon segment transcription to avoid retaining sensitive voice data.
