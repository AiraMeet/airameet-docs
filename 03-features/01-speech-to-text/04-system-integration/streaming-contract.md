# 04. System Integration — Streaming Contract

## Purpose and status

Define the streaming lifecycle and the decisions required before client/backend/AI implementation. Current design is **proposed**; exact protocol and message schemas are TBD.

## Scope and prerequisites

Capture-to-transcript path for a participant stream. Requires auth/meeting membership contract, selected model behavior, approved [architecture](../01-planning-design/system-architecture.md), and privacy rules.

## Proposed lifecycle

1. Client obtains microphone permission and opens an authenticated audio connection for an active meeting.
2. Backend authenticates the user, validates meeting membership, creates a server-owned stream context and assigns a stream ID.
3. Client sends bounded audio frames with sequence information. It does not include trusted user identity.
4. Backend validates size/sequence, applies backpressure and forwards audio plus server-derived meeting/participant context to AI.
5. AI emits partial/final transcript updates with segment identity and time offsets. Backend validates stream association and broadcasts only to authorized meeting members.
6. Final segments are persisted idempotently. On mute, stop, disconnect, timeout or meeting end, services flush/discard according to the agreed policy and release buffers.

## Interface fields to standardize

Audio input: protocol/version, audio format, sample rate, channel count, framing duration/bytes, stream ID, monotonically increasing sequence, optional client capture time, end-of-stream signal. Identity comes from backend auth/context, never client metadata.

Transcript update: meeting/stream/segment IDs; revision or sequence; partial/final flag; text; start/end offsets and timebase; optional confidence with defined meaning; language/script metadata only if the model can support it reliably. Backend broadcast adds display identity from trusted participant records. History API exposes finalized records only unless product requirements say otherwise.

Specify maximum frame/message size, idle timeout, queue limit, retry/resume semantics, duplicate handling, error codes, clock source, and ordering before implementation. Do not treat illustrative examples as final schemas.

## Experimental defaults

Source drafts proposed raw mono 16 kHz PCM, client frames around 250 ms, recognition windows around 1.5 s with 300 ms overlap, VAD and overlapping-context deduplication. These are starting values only. PCM byte rate should inform network sizing; sample rate conversion quality and ownership must be explicit. Partial outputs need stable IDs/revisions so clients replace hypotheses rather than append duplicates. Final boundary alignment should use token/time alignment or a tested sequence-alignment strategy; naive longest-common-substring text matching can fail on repeated words and code-switching.

## Implementation steps

1. Publish versioned protocol schemas and compatibility policy.
2. Implement auth, admission limits, stream context and cancellation in backend.
3. Implement bounded AI queues, VAD/window policy and model adapter.
4. Implement event ordering, partial replacement/final idempotency, persistence handoff and client reconnect behavior.
5. Add contract tests for framing, malformed messages, backpressure, reconnect, final flush, ordering and authorization.

## Tools and technologies

WebSocket/WSS, SignalR for transcript delivery, and gRPC or internal WebSocket for AI forwarding were mentioned in source planning. Choose based on existing platform patterns and measured needs. WebRTC conferencing audio and the STT capture path are separate concerns unless a validated media integration supersedes the proposed parallel path.

## Deliverables and acceptance criteria

Deliver: versioned audio/event schemas, sequence diagrams, error/recovery table and contract tests. Accept when supported clients and services interoperate against the same schema; identity is server-bound; partial revisions and final records are ordered/idempotent; queue limits and disconnect behavior are tested.

## Dependencies and open questions

See [requirements](../01-planning-design/requirements.md) and [attribution](speaker-attribution.md). Decide transport, authentication header/token handling for browser WebSockets, format, sample rate, timestamps/timebase, chunking, VAD ownership, partial semantics, reconnect/resume, flush rules and delivery guarantees.
