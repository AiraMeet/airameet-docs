# Requirements Specification: Speech-to-Text Feature

## 1. Requirements Framework & Taxonomy

This document outlines the formal technical and operational requirements for the **Speech-to-Text & Speaker Attribution** feature. Requirements follow the **MoSCoW** prioritization framework (Must Have, Should Have, Could Have, Won't Have) and the **EARS** (Easy Approach to Requirements Syntax) format.

---

## 2. Functional Requirements (FR)

### 2.1 Audio Ingestion & Capture
| Req ID | Priority | Requirement Statement | Acceptance Criteria |
| :--- | :---: | :--- | :--- |
| `FR-STT-001` | **Must Have** | When a user un-mutes their microphone, the client `AudioWorklet` shall capture raw audio, downsample it to 16 kHz Mono 16-bit linear PCM, and stream it to the backend via WebSocket. | Emitted frames are 16 kHz single-channel Int16Array dispatched every 250ms ($\pm 15\text{ms}$). |
| `FR-STT-002` | **Must Have** | While a user is muted, the client shall cease transmitting audio frames over the WebSocket to conserve client bandwidth and server compute. | Frame transmission rate drops to zero packets per second upon mute toggle. |
| `FR-STT-003` | **Must Have** | The backend shall accept binary WebSocket connections only after successfully validating the participant's JWT Bearer token and room authorization. | Unauthenticated or expired connections receive `401 Unauthorized` and are closed immediately. |

### 2.2 Voice Activity Detection & Chunking
| Req ID | Priority | Requirement Statement | Acceptance Criteria |
| :--- | :---: | :--- | :--- |
| `FR-STT-004` | **Must Have** | The AI ingestion engine shall apply Voice Activity Detection (VAD) to incoming audio frames and filter out non-speech silence and static background noise. | Non-speech audio frames consume zero deep learning forward passes. |
| `FR-STT-005` | **Must Have** | The AI engine shall buffer speech frames into sliding windows of 1.0–2.0 seconds with 200–400ms overlap before triggering neural network inference. | Nominal chunk duration defaults to 1.5 seconds with 300ms overlap. |

### 2.3 Deep Learning Transcription & Deduplication
| Req ID | Priority | Requirement Statement | Acceptance Criteria |
| :--- | :---: | :--- | :--- |
| `FR-STT-006` | **Must Have** | The deep learning model shall output partial (interim) transcript hypotheses as speech progresses, followed by finalized transcript segments upon speech pauses. | Partial segments flagged `is_final = false`; completed utterances flagged `is_final = true`. |
| `FR-STT-007` | **Must Have** | The text post-processor shall apply algorithmic prefix deduplication on overlapping chunk boundaries to prevent repeated or skipped words. | Synthetic test suite confirms 0.0% repeated tokens across chunk overlaps. |
| `FR-STT-008` | **Should Have** | The transcript normalizer shall apply automatic punctuation, capitalization, and formatting of spoken numbers and currency symbols. | Lowercase raw tokens transformed into punctuated, capitalized sentences. |

### 2.4 Speaker Attribution & Persistence
| Req ID | Priority | Requirement Statement | Acceptance Criteria |
| :--- | :---: | :--- | :--- |
| `FR-STT-009` | **Must Have** | The backend shall derive speaker attribution exclusively from the authenticated session context, permanently binding the user's `Guid` to the transcript segment. | Discard any client-supplied `user_id` payload; associate `TranscriptSegment.UserId` strictly with verified session identity. |
| `FR-STT-010` | **Must Have** | The backend shall persist all finalized transcript segments (`is_final = true`) into SQL Server with meeting ID, sequence number, start/end millisecond offsets, and text. | EF Core unit-of-work writes record with monotonically increasing sequence number per room. |
| `FR-STT-011` | **Must Have** | The backend SignalR hub shall broadcast live transcript events to all connected room participants with speaker name, avatar URL, and text. | Event received and rendered by all room participants within 200ms of server emission. |

---

## 3. Non-Functional Requirements (NFR)

### 3.1 Latency & Performance SLA
- **`NFR-STT-001` Visible Latency**: The total end-to-end visible latency from the moment a word is spoken into the microphone until it appears in the client transcript drawer shall not exceed **3.0 seconds (p95)** under normal operational network conditions.
- **`NFR-STT-002` Real-Time Factor (RTF)**: The inference engine shall maintain an RTF $\le 0.25$ on production GPU hardware (processing 1.0 second of audio in $\le 250\text{ms}$).
- **`NFR-STT-003` Concurrency**: A single GPU worker instance (16 GB VRAM) shall support at least **16 concurrent active speaker streams** without dropping frames.

### 3.2 Accuracy & Quality Targets
- **`NFR-STT-004` Relative Word Error Rate (WER) Improvement**: The fine-tuned speech model shall achieve a **$\ge 15\%$ relative WER improvement** over the zero-shot pretrained baseline on the independent meeting test set.
- **`NFR-STT-005` Acoustic Robustness**: The system shall maintain acceptable transcript intelligibility at Signal-to-Noise Ratios (SNR) as low as 10 dB.

### 3.3 Security & Data Privacy
- **`NFR-STT-006` Memory Transient Processing**: Raw audio frames must be processed in volatile memory and overwritten immediately upon chunk completion. No raw audio files shall be written to disk unless explicit cloud recording is enabled by the meeting host.
- **`NFR-STT-007` Transport Encryption**: All external client-to-server traffic must be encrypted via TLS 1.3 over WSS (WebSockets) and HTTPS.
- **`NFR-STT-008` Room Isolation**: Meeting participants shall strictly be prevented from subscribing to or receiving transcript broadcasts from meetings they are not actively authorized to attend.

---

## 4. Edge Cases & Boundary Handling

| Edge Case Scenario | Potential Failure Mode | Mandated System Behavior |
| :--- | :--- | :--- |
| **Simultaneous Speech (Cross-Talk)** | Multiple participants speak at the exact same moment. | Because each participant transmits a separate dedicated audio stream, both streams are processed in parallel by the STT worker and rendered as distinct, interleaved speaker blocks based on their timestamps. |
| **Whispering / Low Volume** | VAD fails to classify faint speech as active voice. | Configurable VAD energy threshold; client-side automatic gain control (AGC) enabled in browser `getUserMedia` constraints. |
| **Continuous Unbroken Speech (> 15s)** | Sliding window grows excessively; model runs out of context memory or hallucinates. | Enforce hard chunk boundary cutoff at 15.0 seconds. Commit current segment as `is_final = true` and reset buffer state. |
| **Sudden Client Disconnection** | Incomplete audio buffer stuck in server memory. | Server WebSocket close event triggers immediate buffer flush, running inference on remaining frames and committing final transcript. |
| **Zero Audio Input (Dead Air)** | Model generates phantom repetitive tokens or hallucinations. | VAD gate suppresses silent chunks entirely, bypassing the neural network. |
