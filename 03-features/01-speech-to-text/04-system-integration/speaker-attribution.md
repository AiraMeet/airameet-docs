# 04. System Integration — Speaker Attribution

## Purpose and decision status

Specify how recognized speech is associated with a meeting participant. **Established design constraint:** attribution is derived from authenticated backend session context; the client cannot choose the speaker identity. Session-bound attribution is the intended MVP approach. Acoustic diarization is not part of the documented MVP scope.

## Scope and prerequisites

Applies from audio connection admission through AI request context, transcript event and persisted record. Requires trusted JWT/identity claims, meeting membership checks, participant lifecycle and [streaming contract](streaming-contract.md).

## Technical approach

1. Authenticate the connection and validate that the authenticated user is an active participant in the requested meeting.
2. Create a server-side stream context bound to user ID, meeting ID and connection/stream ID. Do not accept identity from audio payloads.
3. Forward server-derived context to AI over a protected internal interface. AI returns the context/stream correlation token; backend rejects mismatches.
4. Resolve display name/avatar through backend participant data. Persist stable participant ID with final transcript; presentation metadata can change independently.
5. Authorize every transcript subscription, retrieval and broadcast group membership. Disconnect/revoke streams when participant leaves or meeting ends.

## Ownership and security controls

Backend owns identity validation, room authorization, binding, event audience and persisted attribution. AI recognizes audio and correlates outputs to opaque server context; it is not an identity authority. Frontend displays server-provided attribution and never submits a `user_id` as a source of truth.

Threat cases: forged client identity; expired/revoked token; user not in meeting; cross-meeting stream reuse; stale connection after leave; mismatched AI response correlation; reconnect creating duplicate active streams; unauthorized history access. Test each as deny-by-default.

## Diarization boundary

Separate per-participant device streams support deterministic session attribution for the person providing that stream. They do **not** identify multiple physical people sharing one microphone, nor prove that the authenticated participant is the only voice in their environment. If shared-room hardware becomes a requirement, assess acoustic diarization as a separate capability with privacy, biometric, accuracy and latency review. Avoid numerical claims comparing diarization accuracy/cost unless measured for the use case.

## Tools and technologies

JWT claims and ASP.NET Core authorization are described by the existing project material. Exact claim mapping, WebSocket token transport, internal envelope and participant storage schema must follow platform standards and be documented before implementation.

## Deliverables

Authorization sequence and threat model; server-owned stream context; internal correlation contract; security test cases; audit-safe logs that exclude raw audio and sensitive transcript data.

## Acceptance criteria

Spoofed/missing identity is rejected; valid users can only stream/subscribe within authorized meetings; AI response correlation cannot change attribution; transcript history persists stable participant identity; leave/revocation closes access; no raw audio or token secrets are logged.

## Dependencies and open questions

Depends on identity claims, meeting membership lifecycle, token handling and transcript access policy. Define multiple devices per participant, reconnect behavior, guest identity, account deletion effects, host controls and whether/how to indicate uncertain attribution when background voices are present.
