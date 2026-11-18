# Technology Stack Specification

This document details the complete technology matrix for AiraMeet, documenting **Confirmed Decisions**, **Architectural Rationale**, and areas marked **[To Be Decided - TBD]** or **[Assumption]**.

---

## 1. Stack Matrix Overview

| Layer / Component | Technology Selected | Version / Spec | Status | Rationale |
| :--- | :--- | :--- | :--- | :--- |
| **Backend Framework** | ASP.NET Core Web API | .NET 8 / 9 | **Confirmed** | High-throughput async runtime, enterprise Clean Architecture, native dependency injection, robust security ecosystem. |
| **Backend Language** | C# | 12 / 13 | **Confirmed** | Strongly typed, memory-safe, exceptional performance with Span<T> and Memory<T> for high-frequency operations. |
| **ORM / Data Access** | Entity Framework Core | 8.0+ | **Confirmed** | Code-First migrations, Linq querying, enterprise Identity integration, robust SQL Server support. |
| **Primary Database** | Microsoft SQL Server | 2022+ / Azure SQL | **Confirmed** | ACID compliance, transactional integrity, JSON column indexing, enterprise security. |
| **Cache & State Store** | Redis | 7.2+ | **Confirmed** | High-speed in-memory caching, distributed session state, SignalR message backplane. |
| **Frontend Framework** | Angular | 17+ (Standalone) | **Confirmed** | Enterprise structure, native TypeScript integration, reactive state management (RxJS / Signals), built-in security. |
| **Frontend Language** | TypeScript | 5.3+ | **Confirmed** | Type safety across client state, strict null checks, contract parity with backend models. |
| **Frontend Tooling** | Vite / Angular CLI (esbuild) | Latest | **Confirmed** | Ultra-fast Hot Module Replacement (HMR) and optimized build bundles. |
| **AI / ML Service** | Python | 3.11+ | **Confirmed** | Industry standard for deep learning, PyTorch integration, broad scientific compute support. |
| **AI Web Framework** | FastAPI | 0.110+ | **Confirmed** | Asynchronous Python framework with ASGI (Uvicorn), Pydantic request validation, high-speed streaming capabilities. |
| **ML Inference Engine** | PyTorch / CTranslate2 / TensorRT | Latest | **Proposed** | PyTorch for training; CTranslate2 or ONNX Runtime for low-latency, memory-efficient GPU serving. |
| **Real-Time Video/Audio** | WebRTC (Selective Forwarding Unit - SFU) | RFC 8829 | **[Assumption]** | LiveKit or Mediasoup evaluated as the SFU bridge for multi-party audio/video distribution. |
| **Real-Time Data / Transcripts** | WebSockets / ASP.NET SignalR | W3C Standard | **Confirmed** | Bidirectional low-latency pipeline for streaming audio chunks and broadcasting live transcripts. |
| **Containerization** | Docker & Docker Compose | Docker Engine 25+ | **Confirmed** | Isolated multi-container local and staging environments. |
| **Orchestration** | Kubernetes (K8s) | 1.28+ | **[TBD]** | Production container orchestration with GPU node scheduling. |

---

## 2. Detailed Component Breakdown

### 2.1 Backend Core (ASP.NET Core)
The backend is structured according to **Clean Architecture** principles across four core projects:
- **`AiraMeet.Domain`**: Pure domain entities, value objects, domain events, and core business rules (no external framework dependencies).
- **`AiraMeet.Application`**: CQRS features, DTOs, service abstractions, validation logic, and interfaces (`IIdentityService`, `IJwtTokenService`, `IRefreshTokenRepository`).
- **`AiraMeet.Infrastructure`**: Concrete database contexts (`ApplicationDbContext`), Entity Framework Core mappings, ASP.NET Core Identity integration, and external adapter implementations.
- **`AiraMeet.Api`**: API controllers (`AuthController`), middleware (Global Exception Handling), OpenAPI configuration, and dependency injection wiring.

#### Authentication & Authorization Architecture
- **Identity Framework**: ASP.NET Core Identity with custom `ApplicationUser` using `Guid` primary keys.
- **Access Tokens**: Short-lived JSON Web Tokens (JWT) with configurable expiration (default: 15 minutes).
- **Refresh Tokens**: Cryptographically secure 64-byte random tokens stored in SQL Server with rotation on use, expiry tracking, and revocation tracking.

```csharp
// Current Identity Configuration Snapshot
builder.Services.AddIdentityCore<ApplicationUser>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredLength = 6;
    options.User.RequireUniqueEmail = true;
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
})
.AddRoles<IdentityRole<Guid>>()
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();
```

---

### 2.2 Frontend Application (Angular & TypeScript)
- **Architecture**: Single Page Application (SPA) using Angular Standalone Components, dependency injection, and Reactive Signals.
- **State Management**: Angular Signals for fine-grained local reactivity, combined with RxJS for real-time WebSocket event streams.
- **Styling & UI**: Tailwind CSS coupled with modern modular components and RTL (Right-to-Left) Arabic localization support.
- **Audio Capture**: Browser Web Audio API (`AudioContext`, `MediaStreamAudioSourceNode`, `AudioWorkletNode`) streaming uncompressed 16 kHz mono PCM frames directly into WebSocket connections.

---

### 2.3 AI Services (Python & FastAPI)
- **Runtime**: Python 3.11 running on Debian Linux with CUDA 12.x drivers.
- **Microservice Framework**: FastAPI deployed with `uvicorn` / `gunicorn` workers.
- **Deep Learning Ecosystem**:
  - `torch` and `torchaudio` for tensor operations.
  - `transformers` and `datasets` (Hugging Face) for model loading and tokenization.
  - `evaluate` and `jiwer` for real-time evaluation of WER and CER.
  - `ctranslate2` for production model quantization (INT8/FP16) and accelerated inference.
  - `silero-vad` or `webrtcvad` for client/server voice activity detection.

---

## 3. Communication Protocols

| Source | Destination | Protocol | Payload Type | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Angular Client** | **ASP.NET API** | HTTPS (REST) | JSON | Authentication, meeting creation, profile management, history retrieval. |
| **Angular Client** | **ASP.NET API** | WSS (WebSocket / SignalR) | Binary / JSON | Audio chunk streaming (Client $\rightarrow$ Server), Live transcript broadcast (Server $\rightarrow$ Client). |
| **ASP.NET API** | **FastAPI AI** | gRPC / Internal HTTP/2 | Protobuf / Binary | Forwarding audio chunks for STT inference, receiving partial/final transcripts. |
| **Meeting Clients** | **WebRTC SFU** | SRTP / DTLS | Opus / VP8 / H.264 | Peer audio and video media distribution. |

---

## 4. Confirmed Decisions vs. Open Architectural Decisions

### Confirmed Decisions
1. **Separation of Concerns**: ASP.NET Core manages user sessions, room authorization, and persistence. Python FastAPI is dedicated to deep learning execution.
2. **Session-Bound Identity**: Spoken audio is tied directly to the authenticated session ID on the backend. No client-side audio payload may claim an arbitrary `user_id`.
3. **Database Selection**: SQL Server is the system of record for all persistent metadata, users, meetings, and transcripts.

### Proposed / Open Architectural Decisions (TBD)
1. **SFU Integration vs Direct WebSocket Streaming**:
   - *Option A*: Clients send audio to SFU for conferencing, and SFU taps participant streams to forward to AI service.
   - *Option B*: Clients stream conferencing audio via SFU, while simultaneously streaming a low-bandwidth 16 kHz mono PCM track to ASP.NET Core via WebSockets for transcription.
   - *Recommendation*: **Option B** for Phase 1 MVP to minimize SFU dependency and simplify debugging.
2. **Inter-Service Protocol (Backend $\leftrightarrow$ AI)**:
   - *Option A*: Binary WebSockets.
   - *Option B*: gRPC (HTTP/2 with Protobuf).
   - *Recommendation*: gRPC for deterministic typing and high throughput under concurrent load.
