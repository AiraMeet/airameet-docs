# API Documentation & Contracts

## 1. Overview & Protocol Standard

AiraMeet employs a dual-protocol architecture:
1. **RESTful HTTP APIs**: For stateless resource management, authentication, user profiles, meeting configurations, and historical transcript queries. All payloads use UTF-8 JSON.
2. **WebSockets / SignalR Hubs**: For bidirectional, low-latency streaming of raw audio frames (Client $\rightarrow$ Server) and broadcast of real-time transcription events (Server $\rightarrow$ Clients).

All authenticated endpoints require an `Authorization: Bearer <JWT>` header.

---

## 2. Authentication Endpoints (`/api/auth`)

### 2.1 Register User
- **Method**: `POST`
- **Route**: `/api/auth/register`
- **Auth**: Anonymous

#### Request Body
```json
{
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com",
  "password": "Password123!"
}
```

#### Response (`200 OK`)
```json
{
  "user": {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "email": "john.doe@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "profilePictureUrl": null
  },
  "tokens": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "4a7b9c1d...",
    "expiresIn": 900
  }
}
```

---

### 2.2 Login User
- **Method**: `POST`
- **Route**: `/api/auth/login`
- **Auth**: Anonymous

#### Request Body
```json
{
  "email": "john.doe@example.com",
  "password": "Password123!"
}
```

#### Response (`200 OK`)
Same payload as Register response containing `user` object and `tokens`.

---

### 2.3 Rotate Refresh Token
- **Method**: `POST`
- **Route**: `/api/auth/refresh`
- **Auth**: Anonymous

#### Request Body
```json
{
  "refreshToken": "4a7b9c1d..."
}
```

#### Response (`200 OK`)
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "8f3e2a1b...",
  "expiresIn": 900
}
```

---

### 2.4 Get Current User Profile
- **Method**: `GET`
- **Route**: `/api/auth/me`
- **Auth**: `Bearer <JWT>`

#### Response (`200 OK`)
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "john.doe@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "profilePictureUrl": "https://assets.airameet.com/avatars/john.png"
}
```

---

## 3. Meeting Management Endpoints (`/api/meetings`)

### 3.1 Create Meeting
- **Method**: `POST`
- **Route**: `/api/meetings`
- **Auth**: `Bearer <JWT>`

#### Request Body
```json
{
  "title": "AiraMeet Architecture Review",
  "description": "Weekly architectural alignment on STT pipeline",
  "scheduledStartTime": "2026-11-25T10:00:00Z"
}
```

#### Response (`201 Created`)
```json
{
  "id": "8b512c98-e734-4bc3-9562-e6e2213e2f10",
  "meetingCode": "air-482-kmv",
  "title": "AiraMeet Architecture Review",
  "description": "Weekly architectural alignment on STT pipeline",
  "status": "Scheduled",
  "hostId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "createdAtUtc": "2026-11-17T15:30:00Z"
}
```

---

### 3.2 Get Meeting Details
- **Method**: `GET`
- **Route**: `/api/meetings/{idOrCode}`
- **Auth**: `Bearer <JWT>`

#### Response (`200 OK`)
```json
{
  "id": "8b512c98-e734-4bc3-9562-e6e2213e2f10",
  "meetingCode": "air-482-kmv",
  "title": "AiraMeet Architecture Review",
  "status": "Active",
  "participantCount": 4,
  "actualStartTimeUtc": "2026-11-17T15:32:10Z"
}
```

---

### 3.3 Retrieve Historical Meeting Transcripts
- **Method**: `GET`
- **Route**: `/api/meetings/{id}/transcripts`
- **Auth**: `Bearer <JWT>`

#### Query Parameters
- `page`: default `1`
- `pageSize`: default `100`

#### Response (`200 OK`)
```json
{
  "meetingId": "8b512c98-e734-4bc3-9562-e6e2213e2f10",
  "totalSegments": 128,
  "segments": [
    {
      "id": "11ab34cd-56ef-7890-abcd-ef1234567890",
      "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "speakerName": "John Doe",
      "sequenceNumber": 1,
      "startOffsetMs": 1420,
      "endOffsetMs": 4850,
      "text": "Welcome everyone to our speech-to-text architectural review.",
      "confidence": 0.962,
      "isFinal": true,
      "timestamp": "2026-11-17T15:32:15Z"
    }
  ]
}
```

---

## 4. Real-Time Streaming & WebSocket Protocols

### 4.1 Client Audio Stream Ingestion (Client $\rightarrow$ Backend)
- **Protocol**: `WSS (Secure WebSocket)`
- **Route**: `/ws/meetings/{meetingId}/audio-stream`
- **Handshake Authentication**: `Authorization: Bearer <JWT>` header or ticket query parameter.

#### Handshake Security Validation
1. Backend validates JWT signature, claims, and expiration.
2. Extracts `sub` claim as `UserId`.
3. Validates that `UserId` is an authorized active participant in `meetingId`.
4. Creates in-memory session bridge linking this specific WebSocket connection ID directly to `UserId`.

#### Message Payloads (Binary Audio Frames)
- Format: Uncompressed 16-bit signed integer linear PCM (Little-Endian).
- Channels: 1 (Mono).
- Sampling Rate: 16,000 Hz (16 kHz).
- Frequency: Client transmits binary frames every 250 milliseconds (4,000 samples = 8,000 bytes per frame).

---

### 4.2 Live Transcript Broadcast (Backend $\rightarrow$ Meeting Clients)
- **Protocol**: `SignalR / WebSocket Hub`
- **Hub Route**: `/hubs/meeting-transcripts?meetingId={meetingId}`

#### Event: `ReceiveTranscriptSegment`
Emitted by the server to all room participants whenever interim or final speech chunks are processed.

```json
{
  "segmentId": "d34e5f67-890a-bcde-f123-4567890abcde",
  "meetingId": "8b512c98-e734-4bc3-9562-e6e2213e2f10",
  "speaker": {
    "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "displayName": "John Doe",
    "avatarUrl": "https://assets.airameet.com/avatars/john.png"
  },
  "sequenceNumber": 14,
  "startOffsetMs": 45200,
  "endOffsetMs": 48100,
  "text": "We have validated the Whisper baseline against our test dataset.",
  "isFinal": true,
  "confidence": 0.945
}
```

#### Event: `ReceiveParticipantVoiceState`
Emitted when participant voice activity changes.

```json
{
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "isSpeaking": true,
  "isMuted": false
}
```

---

## 5. Internal Backend $\leftrightarrow$ AI Service Contract (FastAPI)

- **Protocol**: gRPC / HTTP/2 or Private WebSocket
- **Endpoint**: `/ws/ai/v1/transcribe`

### Inbound Frame from ASP.NET Core
```protobuf
message AudioChunkPacket {
  string session_id = 1;
  string meeting_id = 2;
  string user_id = 3;
  int64 sequence_number = 4;
  int64 timestamp_offset_ms = 5;
  bytes pcm_audio = 6;
  bool is_stream_end = 7;
}
```

### Outbound Response from FastAPI
```protobuf
message TranscriptionResultPacket {
  string session_id = 1;
  string meeting_id = 2;
  string user_id = 3;
  int64 sequence_number = 4;
  int64 start_offset_ms = 5;
  int64 end_offset_ms = 6;
  string recognized_text = 7;
  bool is_final = 8;
  float confidence = 9;
}
```
