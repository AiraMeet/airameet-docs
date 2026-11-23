# 01. Planning & Design — Requirements

## 1. Traceable Requirements Overview

This document specifies the testable **Functional Requirements (FR)**, **Non-Functional Requirements (NFR)**, and **Edge Cases** for Feature 01. All requirements follow the EARS (Easy Approach to Requirements Syntax) framework and map directly to tasks and verification test cases.

---

## 2. Functional Requirements (FR)

| ID | Priority | Requirement Statement | Verification Method | Owning Area |
| :--- | :---: | :--- | :--- | :--- |
| **FR-STT-001** | **Must** | When a participant un-mutes in an active meeting, the client `AudioWorklet` shall capture microphone audio, downsample it to 16 kHz Mono 16-bit PCM, and stream 250ms chunks over WebSocket. | Automated browser test measuring frame size (8,000 bytes) and 250ms dispatch interval. | Frontend |
| **FR-STT-002** | **Must** | When a participant mutes or exits, the client shall immediately stop transmitting audio and send an end-of-stream signal. | Stream monitor confirms zero frame dispatch after mute event. | Frontend |
| **FR-STT-003** | **Must** | The backend shall authenticate every incoming audio WebSocket connection using Bearer JWT, verifying user identity and active meeting membership before accepting frames. | Security test verifies immediate rejection (HTTP 401/403) of invalid, expired, or non-member tokens. | Backend |
| **FR-STT-004** | **Must** | The backend shall derive speaker identity exclusively from authenticated session context and inject verified `user_id` into the AI stream; no client audio metadata shall be trusted for identity. | Threat injection test confirms forged client claims are completely ignored. | Backend |
| **FR-STT-005** | **Must** | The AI pipeline shall filter non-speech audio using Voice Activity Detection (Silero VAD) to eliminate background noise and prevent model hallucinations. | VAD evaluation test verifies $\ge 95\%$ suppression on pure silence/noise audio files. | AI |
| **FR-STT-006** | **Must** | The ASR model shall transcribe Egyptian Arabic, English, and Arabic-English code-switching without dropping Latin-script technical words. | Evaluation against bilingual test split demonstrates correct transcription of both scripts. | AI |
| **FR-STT-007** | **Must** | The AI service shall emit interim hypotheses (`is_final: false`) during continuous speech and finalized segments (`is_final: true`) upon silence detection ($\ge 500\text{ms}$). | Streaming contract test verifies interim revision updates and final commitment flags. | AI + Backend |
| **FR-STT-008** | **Must** | The pipeline shall prevent word duplication or loss across sliding window chunk boundaries using longest common substring token alignment. | Boundary regression corpus confirms zero duplicate words across concatenated final segments. | AI |
| **FR-STT-009** | **Must** | When diarization is enabled, the system shall segment multi-speaker audio, detect speaker change points, and associate ASR segments with corresponding speaker IDs. | RTTM evaluation benchmark measuring speaker segmentation and attribution accuracy. | AI |
| **FR-STT-010** | **Must** | The backend shall persist finalized transcript segments into SQL Server with meeting ID, participant ID, user ID, sequence number, start/end millisecond offsets, and text. | EF Core integration test queries persisted segments and verifies ordering and schema validity. | Backend |
| **FR-STT-011** | **Must** | The backend shall multicast live transcript segments via SignalR strictly to authorized participants in the corresponding meeting room. | Multi-room broadcast test confirms no transcript leakage across distinct meeting rooms. | Backend |
| **FR-STT-012** | **Should**| The client interface shall render real-time captions with speaker names, supporting proper bidirectional text alignment (RTL Arabic and LTR English). | UI visual regression test validates rendering of pure Arabic, pure English, and mixed sentences. | Frontend |

---

## 3. Non-Functional Requirements (NFR)

| ID | Category | Target Metric / Constraint | Verification Strategy | Status |
| :--- | :--- | :--- | :--- | :---: |
| **NFR-STT-001** | **Latency** | End-to-end visible latency from speech to subtitle render: $\mathbf{\le 3.0\text{ seconds}}$ (nominal: 1.8s, p95: 2.6s). | Time-synchronized audio injection test measuring client render timestamp vs audio timestamp. | **Target** |
| **NFR-STT-002** | **Throughput** | Real-Time Factor (RTF) $\mathbf{\le 0.15}$ on production GPU (NVIDIA RTX 4090 / A10G) using CTranslate2 INT8. | Benchmark script timing inference duration divided by total audio duration. | **Target** |
| **NFR-STT-003** | **Accuracy** | Overall Word Error Rate (WER) $\mathbf{\le 18.0\%}$; Code-Switching WER $\mathbf{\le 22.0\%}$ on held-out test split. | Automated `jiwer` scoring script comparing model outputs to normalized ground truth. | **Target** |
| **NFR-STT-004** | **Improvement** | Fine-tuned model must achieve $\mathbf{\ge 15.0\%}$ relative WER reduction over zero-shot Whisper-Medium baseline. | Comparative scorecard on identical test partition. | **Target** |
| **NFR-STT-005** | **Memory** | AI GPU VRAM footprint $\mathbf{\le 2.5\text{ GB}}$ per inference worker process under full load. | `nvidia-smi` telemetry logging during concurrent streaming tests. | **Target** |
| **NFR-STT-006** | **Concurrency** | AI service must sustain at least **10 concurrent active speaker streams** per GPU worker without queue overflow. | Load test running 10 parallel audio streaming clients. | **Target** |
| **NFR-STT-007** | **Privacy** | Raw audio must be processed in transient memory and **never written to persistent disk** or long-term logs. | Filesystem audit and log inspection following high-volume meeting simulations. | **Constraint** |
| **NFR-STT-008** | **Security** | All network transport must be encrypted with **TLS 1.3** (HTTPS, WSS, internal gRPC mTLS). | Automated security scan and SSL Labs verification. | **Constraint** |

---

## 4. Edge Cases Matrix

| Scenario | Challenge | Architectural Solution & Required Behavior |
| :--- | :--- | :--- |
| **Simultaneous Speakers (Talk-over)** | Audio overlap causes phoneme collision and confused transcription. | In multi-device setups, independent streams isolate audio per user. In single-microphone setups, PyAnnote detects overlapping speech frames and emits separate speaker intervals. |
| **Code-Switching in Fast Speech** | Rapid switching between Arabic and English words (e.g., *"هنعمل Deploy على الـ Cluster"*). | Whisper bilingual BPE tokenizer natively preserves Latin and Arabic token sequences; text normalizer retains English characters without transliteration. |
| **Low Volume / Whispering** | Low acoustic energy risks being cut off by VAD. | VAD sensitivity is tuned to low-energy speech thresholds; browser Web Audio API applies standard software gain control. |
| **Continuous Uninterrupted Monologue** | Utterances $> 15\text{s}$ without a pause can cause Whisper attention drift and memory growth. | Forced segmentation boundary triggered at 15.0 seconds of continuous speech, committing the current segment and starting a new window. |
| **Network Interruption (Disconnect)** | Client drops connection for 2–5 seconds due to poor Wi-Fi. | Backend buffers the current segment, closes connection after 5s timeout; client reconnects, establishes a new stream, and resumes streaming without duplicate final records. |
| **Prolonged Silence / Acoustic Static** | Hallucinations caused by autoregressive decoding on empty audio. | Silero VAD detects non-speech frames and drops them before passing audio to Whisper; Whisper decoder temperature is set to zero with no-speech probability threshold $\ge 0.6$. |
