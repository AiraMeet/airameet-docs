# 04. System Integration — Interface Contracts & Protocols

## 1. Protocol Architecture Overview

AiraMeet employs distinct, purpose-built communication protocols across its integration boundaries:
1. **Client $\rightarrow$ Backend**: Secure WebSocket (`WSS`) for raw binary audio chunk streaming.
2. **Backend $\leftrightarrow$ AI Service**: High-throughput `gRPC` over HTTP/2 with typed Protobuf schemas.
3. **Backend $\rightarrow$ Clients**: `SignalR` WebSocket Hub for multicasting live captions.
4. **Client $\leftrightarrow$ Backend**: RESTful `HTTPS` (JSON) for authentication, meeting lifecycle, and historical transcript queries.

---

## 2. Contract 1: Client Audio Streaming $\rightarrow$ Backend Gateway

- **Protocol**: Secure WebSocket (`WSS`)
- **Route**: `/ws/meetings/{meetingId}/audio-stream`
- **Handshake Header**: `Authorization: Bearer <JWT>`
- **Payload Format**: Uncompressed 16-bit Signed Integer Linear PCM (Little-Endian)
- **Audio Specification**:
  - Sample Rate: 16,000 Hz (16 kHz)
  - Channels: 1 (Single-channel Mono)
  - Bit Depth: 16 bits per sample (2 bytes)
  - Chunk Interval: 250 milliseconds
  - Chunk Size: 4,000 samples = **8,000 bytes per frame**
- **Payload Type**: Binary `ArrayBuffer` (no JSON wrapping). Identity is derived exclusively from the authenticated connection context.

---

## 3. Contract 2: Backend $\leftrightarrow$ AI Service Ingestion (gRPC Protobuf)

- **Protocol**: gRPC over HTTP/2
- **Service Name**: `AudioStreamingService`
- **Definition File**: `audio_streaming.proto`

```protobuf
syntax = "proto3";

package airameet.ai.v1;

service AudioStreamingService {
  // Bidirectional streaming RPC: streams audio chunks in, streams transcriptions out
  rpc StreamAudio (stream AudioChunkPacket) returns (stream TranscriptionResultPacket);
}

message AudioChunkPacket {
  string session_id = 1;          // Unique audio stream identifier
  string meeting_id = 2;          // Target meeting room GUID
  string user_id = 3;             // Authenticated speaker GUID (verified by backend)
  int64 sequence_number = 4;      // Monotonically increasing chunk index
  int64 timestamp_offset_ms = 5;  // Audio offset in milliseconds from stream start
  bytes pcm_audio = 6;            // Raw 16 kHz Mono 16-bit linear PCM bytes
  bool is_stream_end = 7;         // True when participant mutes or leaves
}

message WordToken {
  string word = 1;                // Recognized word token
  int64 start_offset_ms = 2;      // Word start offset in milliseconds
  int64 end_offset_ms = 3;        // Word end offset in milliseconds
  float confidence = 4;           // Token acoustic/language model confidence [0.0, 1.0]
}

message TranscriptionResultPacket {
  string session_id = 1;          // Correlated stream identifier
  string meeting_id = 2;          // Meeting room GUID
  string speaker_id = 3;          // Attributed speaker identifier (user_id or diarization id)
  int64 sequence_number = 4;      // Monotonically increasing segment index
  int64 start_offset_ms = 5;      // Utterance start timestamp in milliseconds
  int64 end_offset_ms = 6;        // Utterance completion timestamp in milliseconds
  string recognized_text = 7;     // Recognized verbatim text
  bool is_final = 8;              // False for interim hypothesis; true for committed segment
  float confidence = 9;           // Segment aggregate confidence score [0.0, 1.0]
  string language = 10;           // Detected language tag: "ar", "en", or "mixed"
  repeated WordToken words = 11;  // Detailed word-level timestamp breakdown
}
```

---

## 4. Contract 3: Backend $\rightarrow$ Client Live Captions (SignalR)

- **Protocol**: SignalR WebSocket Hub
- **Hub Route**: `/hubs/meeting-transcripts?meetingId={meetingId}`
- **Authentication**: Bearer JWT validated during connection handshake.

### Event: `ReceiveTranscriptSegment`
Broadcast to all room members whenever an interim hypothesis or final segment is generated:

```json
{
  "segmentId": "d34e5f67-890a-bcde-f123-4567890abcde",
  "meetingId": "8b512c98-e734-4bc3-9562-e6e2213e2f10",
  "speaker": {
    "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "displayName": "Ahmed Mansour",
    "avatarUrl": "https://assets.airameet.com/avatars/ahmed.png"
  },
  "sequenceNumber": 42,
  "startOffsetMs": 14200,
  "endOffsetMs": 17850,
  "text": "المشروع محتاج نركز على الـ Code-switching أكتر.",
  "isFinal": true,
  "confidence": 0.954,
  "language": "mixed"
}
```

### Event: `ReceiveParticipantVoiceState`
Broadcast when participant voice activity transitions between active and silent:

```json
{
  "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "isSpeaking": true,
  "isMuted": false
}
```

---

## 5. Contract 4: RESTful API Endpoints

### 5.1 Meeting Management
- **`POST /api/meetings`**:
  - Request: `{"title": "Sprint Planning", "description": "Weekly alignment"}`
  - Response (`201 Created`): `{"id": "...", "meetingCode": "air-482-kmv", "status": "Scheduled"}`
- **`GET /api/meetings/{idOrCode}`**:
  - Response (`200 OK`): `{"id": "...", "meetingCode": "air-482-kmv", "title": "...", "status": "Active"}`
- **`POST /api/meetings/{idOrCode}/join`**:
  - Response (`200 OK`): `{"participantId": "...", "role": "Participant", "joinedAt": "..."}`

### 5.2 Historical Transcripts
- **`GET /api/meetings/{id}/transcripts?page=1&pageSize=50`**:
  - Response (`200 OK`):
    ```json
    {
      "meetingId": "8b512c98-e734-4bc3-9562-e6e2213e2f10",
      "totalSegments": 128,
      "page": 1,
      "pageSize": 50,
      "segments": [
        {
          "id": "11ab34cd-56ef-7890-abcd-ef1234567890",
          "userId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "speakerName": "Ahmed Mansour",
          "sequenceNumber": 1,
          "startOffsetMs": 1420,
          "endOffsetMs": 4850,
          "text": "صباح الخير جميعاً، جاهزين نبدأ مراجعة الـ ASR Pipeline؟",
          "confidence": 0.962,
          "isFinal": true,
          "timestamp": "2026-11-23T10:00:15Z"
        }
      ]
    }
    ```
