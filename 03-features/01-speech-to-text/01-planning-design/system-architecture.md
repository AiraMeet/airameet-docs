# 01. Planning & Design — System Architecture

## 1. High-Level Architectural Vision

AiraMeet employs a decoupled, asynchronous, distributed microservice architecture. The platform cleanly isolates low-latency interactive media paths (WebRTC audio/video conferencing) from deep learning inference pipelines (FastAPI Speech-to-Text & Speaker Diarization).

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

    subgraph CoreBackend["Application Core (ASP.NET Core Web API)"]
        AuthModule["Identity & Auth Module<br/>(JWT, Refresh Tokens)"]
        MeetingModule["Meeting Orchestration<br/>(Rooms, Participants, Status)"]
        AudioHub["Audio Streaming Gateway<br/>(WebSocket Handler)"]
        TranscriptHub["Transcript Dispatcher Hub<br/>(SignalR Live Broadcast)"]
        PersistenceWorker["Transcript Persistence Worker<br/>(EF Core Background Channel)"]
    end

    subgraph DataStorage["Persistence & State Tier"]
        SqlServer[("SQL Server<br/>Users, Meetings, Transcripts")]
        RedisStore[("Redis Cluster<br/>Session Registry, Bus")]
    end

    subgraph AIServiceTier["AI Intelligence Tier (Python FastAPI)"]
        gRPCConsumer["Stream Ingestion Gateway<br/>(gRPC Server, Buffer)"]
        VADEngine["Voice Activity Detector (VAD)<br/>(Silero VAD v5)"]
        ASREngine["ASR Inference Engine<br/>(Fine-Tuned Whisper / CTranslate2)"]
        DiarizationEngine["Speaker Diarization Engine<br/>(PyAnnote.audio 3.1)"]
        AlignmentModule["Timestamp Synchronizer<br/>& Deduplication Filter"]
    end

    UI <-->|HTTPS / REST| Ingress
    Worklet -->|WSS Binary PCM Frames| Ingress
    Ingress <-->|HTTPS / REST| AuthModule
    Ingress <-->|HTTPS / REST| MeetingModule
    Ingress <-->|WSS Audio| AudioHub
    Ingress <-->|SignalR WSS| TranscriptHub

    AudioHub <--> RedisStore
    TranscriptHub <--> RedisStore
    AuthModule <--> SqlServer
    MeetingModule <--> SqlServer
    PersistenceWorker --> SqlServer

    AudioHub -->|Internal gRPC Stream| gRPCConsumer
    gRPCConsumer --> VADEngine
    VADEngine -->|Speech Chunks| ASREngine
    gRPCConsumer -->|Continuous Audio| DiarizationEngine
    ASREngine --> AlignmentModule
    DiarizationEngine --> AlignmentModule
    AlignmentModule -->|Attributed Segments| gRPCConsumer
    gRPCConsumer -->|Result Stream| AudioHub
    AudioHub --> PersistenceWorker
    AudioHub --> TranscriptHub
```

---

## 2. Tier-by-Tier Architectural Breakdown

### 2.1 Client Tier (Angular 17+ Standalone)
- **Audio Worklet Subsystem**: Bypasses the UI main thread using an `AudioWorkletProcessor` to tap the microphone `MediaStream`. Captures raw 32-bit Float samples, downsamples to 16 kHz 16-bit signed integer linear PCM, and streams chunks every 250ms over a dedicated WebSocket connection.
- **Reactive State Layer**: Uses Angular Signals and RxJS Subject streams for managing participant connection states, mute/unmute indicators, and live transcript scrolling.
- **Dual-Track Audio**: Conferencing voice is routed via WebRTC SFU; transcription audio is captured directly via the AudioWorklet and transmitted to ASP.NET Core.

### 2.2 Application Core Tier (ASP.NET Core Web API)
- **Security Boundary**: Validates Bearer JWT tokens on every REST endpoint and upgrades WebSockets only after validating token claims and active meeting room participation.
- **Connection Registry**: Maintains active meeting room topologies in Redis memory. When a client opens an audio streaming socket, the server binds the socket connection directly to the authenticated `ApplicationUser.Id` and `MeetingRoom.Id`.
- **Media Ingestion & Forwarding**: Buffers participant audio frames into rolling queues and forwards them to the AI Service Tier via high-speed internal gRPC streams.
- **Transcript Broadcast Hub**: Consumes recognized text tokens from the AI tier, validates permissions, persists finalized segments into SQL Server, and multicasts live updates to all meeting room participants via SignalR.

### 2.3 AI Intelligence Tier (Python FastAPI & PyTorch)
- **Decoupled Compute**: The AI service operates independently of the web application, scaling horizontally based on GPU availability.
- **Voice Activity Detection (VAD)**: Inspects incoming audio frames using Silero VAD v5 to classify frames as speech vs. non-speech (silence, background noise). Non-speech frames are discarded before hitting heavy neural network layers.
- **Streaming Audio Buffer**: Implements a circular sliding window (1.5-second context window with 300ms overlap).
- **Inference Engine**: Executes forward passes on fine-tuned Whisper-Medium using CTranslate2 INT8 quantization on NVIDIA GPUs. Emits **Partial Transcripts** (interim hypothesis) and **Final Transcripts** (committed text with boundary timestamps).
- **Speaker Diarization Engine**: Runs PyAnnote.audio 3.1 on continuous meeting audio to extract speaker change boundaries and cluster voiceprints, aligning speaker IDs to recognized words.

### 2.4 Data & Persistence Tier
- **SQL Server**: Stores relational entities, transactional audit trails, user identities, meeting attendance logs, and permanent meeting transcripts.
- **Redis**: Provides sub-millisecond ephemeral state tracking, token blocklists, active meeting participant rosters, and cross-node message distribution for multi-instance ASP.NET Core clusters.

---

## 3. End-to-End Real-Time Data Flow

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
    NetGateway->>AIService: Internal gRPC Stream: AudioChunkPacket(PCM + verified UserId)

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
    AIService->>AIService: Token boundary alignment & deduplication
    AIService-->>NetGateway: Final Segment: full text, start/end ms, confidence, is_final=true

    Note over Speaker,PeerApps: Phase 5: Asynchronous Persistence & Multicast
    NetGateway->>DB: Async persist TranscriptSegment (UserId, MeetingId, Offsets, Text)
    NetGateway->>Redis: Publish final segment to meeting room channel
    NetGateway-->>SpeakerApp: SignalR Final Segment: is_final=true (Committed text)
    NetGateway-->>PeerApps: SignalR Final Segment: is_final=true (Committed text)
```

---

## 4. End-to-End Latency Budget

To deliver responsive real-time transcription, the total visible transcription latency must remain strictly within **2.0 to 3.0 seconds**.

| Processing Stage | Nominal Latency | Upper Bound (p95) | Technical Optimization |
| :--- | :---: | :---: | :--- |
| **Audio Capture & AudioWorklet** | 250 ms | 300 ms | 250ms periodic buffer dispatch in worker thread. |
| **Client-to-Server Network Transit** | 40 ms | 120 ms | WSS over optimized TCP/TLS socket. |
| **Server Ingestion & VAD Analysis** | 30 ms | 60 ms | Silero VAD running in C++ / TorchScript ONNX. |
| **Sliding Window Accumulation** | 1,200 ms | 1,500 ms | Target window duration for phoneme recognition. |
| **Deep Learning Forward Pass** | 200 ms | 450 ms | CTranslate2 INT8 quantization on NVIDIA GPU. |
| **Deduplication & Alignment** | 15 ms | 35 ms | Longest common substring prefix pruning in Python. |
| **Server-to-Client Broadcast** | 40 ms | 100 ms | ASP.NET Core SignalR hub broadcasting JSON. |
| **Client UI Render & Virtual Scroll** | 10 ms | 30 ms | Angular Reactive Signals with `OnPush` change detection. |
| **Total Cumulative Latency** | **1,785 ms (~1.8s)** | **2,595 ms (~2.6s)** | **Strictly within 3.0s SLA Budget** |
