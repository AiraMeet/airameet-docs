# Phase 8: Integration & Quality Assurance Tasks (WBS)

## 1. Workstream Overview

This document specifies the Quality Assurance (QA) and Integration testing workstream for the Speech-to-Text and Speaker Attribution feature. The tasks span end-to-end automated pipelines, multi-participant load stress harnesses, latency benchmarking, and network fault tolerance tests.

All tasks follow the standardized task schema and three-point PERT estimation model ($E = \frac{O + 4M + P}{6}$).

---

## 2. Tasks Catalog: Quality Assurance & Integration

### `QA-STT-001`: Audio Streaming Test Harness & Frame Validation
- **Task ID**: `QA-STT-001`
- **Task Name**: Build Automated Audio Streaming Test Harness
- **Description**: Construct a Python-based synthetic audio streaming harness that connects to the ASP.NET Core WebSocket gateway, transmits pre-recorded 16 kHz PCM audio files at exact 250ms intervals, and verifies frame ingestion.
- **Priority**: Critical
- **Responsible Department**: Quality Assurance / Test Automation
- **Dependencies**: `BE-STT-003`
- **Implementation Steps**:
  1. Write `tests/harness/audio_streamer.py` using `websockets` and `asyncio`.
  2. Implement packet timer simulating exact 250ms hardware microphone cadence.
  3. Validate byte framing and server acknowledgement.
- **Time Estimates**:
  - Optimistic ($O$): 8h
  - Most Likely ($M$): 12h
  - Pessimistic ($P$): 18h
  - Expected ($E$): $\frac{8 + 4(12) + 18}{6} = 12.33\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Harness reliably transmits audio frames and asserts zero dropped frames under synthetic test execution.
- **Deliverable**: `audio_streamer.py` CLI script and integration tests.
- **Status**: `To Do`

---

### `QA-STT-002`: Multi-Participant Concurrency & Stress Testing
- **Task ID**: `QA-STT-002`
- **Task Name**: Multi-Speaker Concurrency and Load Testing
- **Description**: Simulate 5, 10, and 20 simultaneous active speakers across multiple virtual meeting rooms to test GPU throughput, VRAM limits, and server CPU scaling.
- **Priority**: High
- **Responsible Department**: Quality Assurance / Performance
- **Dependencies**: `QA-STT-001`, `BE-STT-005`
- **Implementation Steps**:
  1. Develop Locust / Python concurrent runner spawning parallel WebSocket streams.
  2. Measure server CPU, RAM, and GPU VRAM saturation.
  3. Verify that all concurrent streams receive timely transcript broadcasts without timeouts.
- **Time Estimates**:
  - Optimistic ($O$): 10h
  - Most Likely ($M$): 16h
  - Pessimistic ($P$): 24h
  - Expected ($E$): $\frac{10 + 4(16) + 24}{6} = 16.33\text{h}$
  - Risk Buffer: 3h
- **Acceptance Criteria**: System sustains 16 concurrent speaker streams on a single GPU with p95 visible latency $\le 3.0$ seconds.
- **Deliverable**: Performance test scripts, Grafana dashboard snapshot, and load report.
- **Status**: `To Do`

---

### `QA-STT-003`: Speaker Attribution & Security Penetration Testing
- **Task ID**: `QA-STT-003`
- **Task Name**: Speaker Attribution Security Verification & Spoofing Defense
- **Description**: Verify that speaker attribution is strictly derived from verified JWT tokens and cannot be spoofed, manipulated, or cross-contaminated across rooms.
- **Priority**: Critical
- **Responsible Department**: Security & QA
- **Dependencies**: `BE-STT-003`, `BE-STT-006`
- **Implementation Steps**:
  1. Execute automated test attempting to inject foreign `user_id` values into audio payloads.
  2. Attempt cross-room eavesdropping by listening to SignalR streams for unauthorized `meetingId`.
  3. Validate SQL Server records match the authenticated `UserId` for every generated segment.
- **Time Estimates**:
  - Optimistic ($O$): 6h
  - Most Likely ($M$): 10h
  - Pessimistic ($P$): 16h
  - Expected ($E$): $\frac{6 + 4(10) + 16}{6} = 10.33\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: 100% of spoofing attempts rejected; zero unauthorized cross-room message leakage.
- **Deliverable**: Security test suite in `tests/security/` and compliance sign-off.
- **Status**: `To Do`

---

### `QA-STT-004`: Boundary Deduplication & Overlap Regression Suite
- **Task ID**: `QA-STT-004`
- **Task Name**: Chunk Boundary Deduplication Automated Test Suite
- **Description**: Validate that words spoken across overlapping 300ms chunk boundaries are never duplicated or omitted in the finalized transcript.
- **Priority**: High
- **Responsible Department**: Quality Assurance / AI
- **Dependencies**: `BE-STT-004`
- **Implementation Steps**:
  1. Synthesize 50 test audio utterances with words intentionally placed on 1.5s boundary splits.
  2. Feed utterances into the streaming pipeline and capture emitted final text.
  3. Compare output with ground truth to assert 0.0% repeated tokens.
- **Time Estimates**:
  - Optimistic ($O$): 6h
  - Most Likely ($M$): 8h
  - Pessimistic ($P$): 14h
  - Expected ($E$): $\frac{6 + 4(8) + 14}{6} = 8.67\text{h}$
  - Risk Buffer: 1.5h
- **Acceptance Criteria**: 0 duplicated words detected across all 50 synthetic boundary test cases.
- **Deliverable**: Automated test suite `tests/test_boundary_dedup.py`.
- **Status**: `To Do`

---

### `QA-STT-005`: Network Fault Tolerance & Jitter Chaos Testing
- **Task ID**: `QA-STT-005`
- **Task Name**: Network Disconnect, Jitter, and Reconnection Test Suite
- **Description**: Emulate unstable networks (packet loss, 100ms jitter, temporary socket disconnects) to verify client reconnection and transcript continuity.
- **Priority**: Medium
- **Responsible Department**: Quality Assurance
- **Dependencies**: `FE-STT-006`
- **Implementation Steps**:
  1. Configure Linux network emulator (`tc netem`) with 10% packet drop and 150ms delay.
  2. Abruptly kill client TCP connections during active speech.
  3. Verify that the Angular client reconnects within 3 seconds and preserves existing transcript history.
- **Time Estimates**:
  - Optimistic ($O$): 8h
  - Most Likely ($M$): 12h
  - Pessimistic ($P$): 18h
  - Expected ($E$): $\frac{8 + 4(12) + 18}{6} = 12.33\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Client recovers automatically after disconnect; no duplicate transcript cards rendered upon reconnection.
- **Deliverable**: Automated chaos test report and regression script.
- **Status**: `To Do`

---

### `QA-STT-006`: End-to-End Multi-User Meeting Scenario Validation
- **Task ID**: `QA-STT-006`
- **Task Name**: End-to-End Meeting Scenario Testing (Playwright/Cypress)
- **Description**: Conduct full user journey testing: User A creates meeting, User B and C join, all three unmute and speak, transcripts render on all screens with speaker labels, meeting concludes, and historical transcripts are queried.
- **Priority**: Critical
- **Responsible Department**: Quality Assurance
- **Dependencies**: `FE-STT-005`, `BE-STT-007`
- **Implementation Steps**:
  1. Write Playwright E2E test launching three headless browser instances.
  2. Feed virtual microphone audio streams to each instance.
  3. Assert live transcript elements appear in the DOM with correct speaker names and timestamps.
  4. End meeting and assert historical transcript API returns all segments.
- **Time Estimates**:
  - Optimistic ($O$): 12h
  - Most Likely ($M$): 18h
  - Pessimistic ($P$): 28h
  - Expected ($E$): $\frac{12 + 4(18) + 28}{6} = 18.67\text{h}$
  - Risk Buffer: 3h
- **Acceptance Criteria**: Full E2E multi-user meeting test passes in CI/CD pipeline.
- **Deliverable**: Playwright E2E test suite in `tests/e2e/meeting_transcription.spec.ts`.
- **Status**: `To Do`
