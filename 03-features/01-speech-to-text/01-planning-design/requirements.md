# 01. Planning & Design — Requirements

## Purpose

Define testable feature behavior and trace acceptance evidence. Priorities and thresholds below are inherited planning requirements, not claims that the system meets them.

## Scope and prerequisites

Applies to audio capture through transcript delivery and storage. Depends on approved product/privacy policy, meeting authorization and a documented client/backend/AI interface. Related design: [system architecture](system-architecture.md); implementation contracts: [streaming](../04-system-integration/streaming-contract.md).

## Functional requirements

| ID | Priority | Requirement | Acceptance evidence | Owner |
|---|---|---|---|---|
| FR-STT-001 | Must | When capture is enabled and microphone access is granted, client captures audio and sends framed audio to the authorized meeting endpoint. | Browser integration test verifies format, sequence/timing and mute behavior against the agreed contract. | Frontend |
| FR-STT-002 | Must | When the participant mutes or stops capture, client stops sending microphone audio. | Frame count stops; backend closes/discards residual stream according to contract. | Frontend + Backend |
| FR-STT-003 | Must | Backend validates authentication and meeting membership before accepting audio or transcript subscriptions. | Unauthorized/expired identity and cross-meeting cases are rejected and audited. | Backend |
| FR-STT-004 | Must | AI pipeline avoids unnecessary recognition work on non-speech where a validated VAD is used. | Evaluation on labeled silence/noise set shows defined suppression and false-negative behavior. | AI |
| FR-STT-005 | Must | Recognition supports bounded streaming context and emits updates without unbounded buffering. | Stream tests verify documented limits, ordering and flush behavior. | AI + Backend |
| FR-STT-006 | Must | Consumers can distinguish replaceable partial hypotheses from finalized segments. | Contract tests verify stable segment IDs/revisions and finalization semantics. | AI + Backend |
| FR-STT-007 | Must | Overlapping audio context does not create duplicated or lost finalized words. | Boundary regression corpus reports insertion/deletion behavior; zero duplication is a proposed gate until validated. | AI |
| FR-STT-008 | Should | Text normalization/punctuation may be applied consistently without losing code-switching or meaning. | Language-specific review set and documented normalization policy. | AI + Product |
| FR-STT-009 | Must | Speaker identity is derived from authenticated meeting session context, never trusted from client audio metadata. | Security tests show identity binding and no spoofing/cross-room leakage. | Backend |
| FR-STT-010 | Must | Final segments persist with meeting, participant, sequence/order, text and time offsets. | Persistence/query integration test verifies schema and ordering. | Backend |
| FR-STT-011 | Must | Authorized participants receive transcript events and UI displays attribution and time reference. | Multi-client end-to-end test verifies audience and rendering. | Backend + Frontend |

## Non-functional requirements and status

| ID | Requirement / initial target | Status and verification |
|---|---|---|
| NFR-STT-001 | End-to-end visible latency p95 ≤ 3 seconds under agreed normal conditions. | **Target**; define workload, clock boundary and environment before measurement. |
| NFR-STT-002 | Real-time factor ≤ 0.25 on production-equivalent hardware. | **Target**; measure per model/runtime and concurrency. |
| NFR-STT-003 | Initial planning assumption: 16 concurrent active participant streams per worker. | **Proposed capacity target**, not a measured limit. Load test and revise. |
| NFR-STT-004 | Fine-tuned model improves relative WER by at least 15% over a chosen baseline on identical held-out data. | **Proposed gate**; review after baseline and dataset audit. |
| NFR-STT-005 | Useful recognition under noisy conditions, including approximately 10 dB SNR. | **Target requiring definition** of noise mix and acceptable quality. |
| NFR-STT-006 | Raw audio is transiently processed and not persisted unless recording policy explicitly authorizes it. | **Required privacy constraint**; verify logs, temp storage and backups. |
| NFR-STT-007 | External traffic uses TLS-protected HTTPS/WSS. | Required; exact TLS policy follows platform security standard. |
| NFR-STT-008 | Meeting authorization prevents unauthorized transcript/audio access. | Required; verify backend authorization on every connection and event path. |

## Edge cases

| Scenario | Required behavior to specify/verify |
|---|---|
| Concurrent speakers | Independent authenticated streams; segments may interleave and carry time references. |
| Low volume / whisper | Define capture processing and VAD behavior; do not promise automatic gain control until browser behavior is validated. |
| Long uninterrupted speech | Apply bounded windows and explicit finalization policy; the prior 15-second cutoff is a proposal requiring product/model validation. |
| Disconnect | Release stream resources; document whether buffered audio is flushed, discarded, or retried. Avoid duplicate final records on reconnect. |
| Silence/noise | Suppress hallucinated output; evaluate VAD false positives/negatives on representative material. |
| Code-switching | Preserve both scripts/languages; avoid forced single-language normalization. |

## Tools and technologies

Current project material references Angular/Web Audio, ASP.NET Core, SignalR, SQL Server/EF Core, and a Python AI service. These are **existing project-context candidates**, not newly approved versions or immutable choices. See [architecture](system-architecture.md).

## Dependencies and open questions

Dependencies: meeting/auth APIs, product privacy policy, labeled test corpus, agreement on audio/event schemas, and deployment environment. Open: absolute clean/noisy WER/CER thresholds; supported browsers; transcript retention and deletion; event delivery guarantees; timestamp reference (meeting-relative vs stream-relative); precise latency budget allocation.
