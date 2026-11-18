# Test Cases Catalog: Speech-to-Text Feature

## 1. Test Suite Summary Matrix

| Category | Suite ID | Test Count | Target Component | Automation Status |
| :--- | :--- | :---: | :--- | :--- |
| **Authentication & Session Attribution** | `TC-AUTH` | 5 | ASP.NET Core API / Security | Automated (xUnit) |
| **Audio Capture & AudioWorklet** | `TC-AUDIO` | 4 | Angular Client / Web Audio API | Automated (Vitest/Karma) |
| **Streaming & WebSocket Gateway** | `TC-STREAM` | 6 | ASP.NET Core & FastAPI Hub | Automated (Integration) |
| **Deduplication & Boundary Handling** | `TC-DEDUP` | 4 | FastAPI AI Engine | Automated (PyTest) |
| **Accuracy & Model Performance** | `TC-ACC` | 5 | PyTorch / CTranslate2 Engine | Automated (Benchmark) |
| **Network Resilience & Fault Tolerance** | `TC-NET` | 4 | Network / WebSocket Layer | Automated (Chaos Harness) |
| **End-to-End Meeting Scenarios** | `TC-E2E` | 3 | Full System Integration | Manual & Cypress/Playwright |

---

## 2. Authentication & Speaker Attribution (`TC-AUTH`)

### TC-AUTH-001: Session-Bound Identity Derivation
- **Description**: Verify that the backend binds incoming audio streams exclusively to the authenticated user identity present in the JWT claims.
- **Preconditions**: User `UserA` (ID: `guid-111`) is authenticated and possesses a valid JWT.
- **Execution Steps**:
  1. Open WebSocket to `/ws/meetings/{meetingId}/audio-stream` with `Bearer <UserA_Token>`.
  2. Send binary audio chunks.
  3. Inspect server-side `TranscriptSegment` record written to database.
- **Expected Result**: The `TranscriptSegment.UserId` equals `guid-111`. Client cannot supply or alter this identifier.

### TC-AUTH-002: Rejection of Spoofed User ID
- **Description**: Verify that client attempts to send an arbitrary `user_id` in binary frames or socket headers are discarded.
- **Preconditions**: User `UserA` connects with valid credentials.
- **Execution Steps**:
  1. Client sends a JSON metadata frame with `{"claimed_user_id": "guid-999"}`.
  2. Speak into microphone.
- **Expected Result**: Backend ignores `claimed_user_id` and persists the transcript under `guid-111`. A security warning is logged.

### TC-AUTH-003: Unauthorized WebSocket Handshake
- **Description**: Verify that unauthenticated or expired tokens fail to establish an audio stream.
- **Preconditions**: Expired JWT or missing `Authorization` header.
- **Execution Steps**:
  1. Attempt WebSocket connection to `/ws/meetings/{meetingId}/audio-stream`.
- **Expected Result**: Server returns `HTTP 401 Unauthorized` and closes socket immediately.

---

## 3. Audio Ingestion & Worklet (`TC-AUDIO`)

### TC-AUDIO-001: 16 kHz Mono PCM Format Verification
- **Description**: Verify that the client AudioWorklet downsamples microphone input to 16 kHz mono signed 16-bit PCM.
- **Preconditions**: Browser audio context running at 48,000 Hz stereo.
- **Execution Steps**:
  1. Initialize `AudioStreamProcessor`.
  2. Feed 48,000 Hz sinusoidal test tone.
  3. Capture emitted array buffer on worker port.
- **Expected Result**: Emitted buffer length corresponds to 16,000 samples/sec, single channel, encoded as `Int16Array` with values in range $[-32768, 32767]$.

### TC-AUDIO-002: Audio Worklet Non-Blocking UI Execution
- **Description**: Verify that heavy main thread UI operations (DOM rendering, chart updates) do not cause buffer underruns in audio capture.
- **Execution Steps**:
  1. Start audio streaming in Angular client.
  2. Trigger CPU-heavy calculation on main thread for 1,000ms.
  3. Measure arrival times of audio frames on server.
- **Expected Result**: Frames arrive at regular 250ms intervals without dropped frames or discontinuities.

---

## 4. Deduplication & Boundary Handling (`TC-DEDUP`)

### TC-DEDUP-001: Boundary Word Deduplication in Overlapping Chunks
- **Description**: Verify that overlapping sliding windows (1.5s window with 300ms overlap) do not duplicate words spoken across chunk boundaries.
- **Preconditions**: Speech audio stream: "We need to finalize the deployment roadmap today."
- **Execution Steps**:
  1. Feed audio through chunking pipeline where the word "deployment" falls directly within the 300ms overlap window between Chunk 1 and Chunk 2.
  2. Record emitted partial and final transcripts.
- **Expected Result**: The word "deployment" appears exactly once in the finalized sentence. Zero duplicate tokens ("deployment deployment") emitted.

### TC-DEDUP-002: Punctuation and Casing Normalization
- **Description**: Verify that normalized text output formats numbers, abbreviations, and sentence casing correctly.
- **Execution Steps**:
  1. Feed audio: "meeting starts at ten am with five participants".
  2. Inspect emitted `NormalizedText`.
- **Expected Result**: Output is formatted as: "Meeting starts at 10:00 AM with 5 participants."

---

## 5. Model Accuracy & Acoustic Benchmarking (`TC-ACC`)

### TC-ACC-001: Test Set Baseline WER Verification
- **Description**: Benchmark original pretrained Whisper / Wav2Vec / NeMo model on the independent test set.
- **Execution Steps**:
  1. Run `benchmark_models.py --dataset data/splits/test --model openai/whisper-small`.
  2. Compute Word Error Rate (WER) and Character Error Rate (CER).
- **Expected Result**: Baseline numerical values recorded for official comparison (Target: WER baseline documented).

### TC-ACC-002: Fine-Tuned Model Relative WER Improvement
- **Description**: Verify that fine-tuned model checkpoint achieves measurable WER improvement over pretrained baseline.
- **Preconditions**: Fine-tuned checkpoint saved at `models/stt-finetuned-best`.
- **Execution Steps**:
  1. Run evaluation suite on identical test set.
  2. Calculate relative error reduction: $\frac{WER_{baseline} - WER_{finetuned}}{WER_{baseline}} \times 100\%$.
- **Expected Result**: Relative error reduction $\ge 15.0\%$ with 0% data leakage verified.

---

## 6. Real-Time Latency & Concurrency (`TC-STREAM`)

### TC-STREAM-001: Visible Transcription Latency Benchmark
- **Description**: Measure elapsed time between client speaking and text rendering on peer screens.
- **Execution Steps**:
  1. Synthesize audio impulse at $T_0$.
  2. Capture timestamp $T_1$ when peer receives final transcript event via SignalR.
  3. Calculate $\Delta T = T_1 - T_0$.
- **Expected Result**: $\Delta T \le 2,500$ ms (p50) and $\le 3,000$ ms (p95).

### TC-STREAM-002: Multi-Speaker Concurrent Load
- **Description**: Test system behavior when 10 meeting participants speak simultaneously in separate streams.
- **Execution Steps**:
  1. Open 10 concurrent WebSocket audio streams for a single meeting room.
  2. Stream 16 kHz PCM audio continuously for 5 minutes.
  3. Monitor CPU, VRAM, and packet drop rate.
- **Expected Result**: Server maintains all 10 streams without dropped connections; GPU inference worker successfully queues and processes streams within latency threshold.

---

## 7. Network Fault Tolerance & Reconnection (`TC-NET`)

### TC-NET-001: Temporary WebSocket Disconnect & Reconnect
- **Description**: Verify that client reconnects cleanly after a 3-second network interruption without losing past transcripts.
- **Execution Steps**:
  1. Start meeting and speak: "Phase one is complete."
  2. Terminate client TCP socket connection abruptly for 3 seconds.
  3. Client reconnects automatically.
  4. Speak: "Moving on to phase two."
- **Expected Result**: Past transcripts remain intact on UI; new utterance is transcribed with correct speaker label and updated sequence number.
