# 04. System Integration — Frontend Ownership

## Purpose

Define client workstream responsibility for capture and transcript experience, with clear server/AI boundaries.

## Scope and prerequisites

Angular meeting UI, browser microphone lifecycle, audio framing, live transcript view and connection states. Requires the approved [streaming contract](streaming-contract.md), design system, supported-browser policy, auth integration and product consent UX.

## Work items and dependencies

| ID | Work item | Dependencies | Acceptance evidence |
|---|---|---|---|
| FE-STT-001 | Existing authentication integration and route/session guards. | Platform identity flow. | Auth expiry and unauthorized state tests; do not rebuild auth solely for STT. |
| FE-STT-002 | Dashboard and create/join meeting flows. | Meeting APIs (likely existing capability). | Validated lifecycle flows and errors. |
| FE-STT-003 | Meeting room controls, mute state and transcript access. | Meeting lifecycle and product UX. | Keyboard/accessibility and state-transition tests. |
| FE-STT-004 | Microphone capture, format conversion and bounded frame sender. | Browser support and audio contract. | Permission, device change, mute, framing and resource cleanup tests. |
| FE-STT-005 | Live transcript UI with speaker/time labels and partial/final semantics. | Backend event schema. | Partial revisions replace correctly; finals remain stable and accessible. |
| FE-STT-006 | Connection status, retry/recovery and error states. | Backend reconnect/resume policy. | Disconnect/reconnect tests without duplicate transcript rendering. |

Original planning estimates (hours, O/M/P): FE-STT-001 10/14/22; 002 8/12/18; 003 12/16/26; 004 12/18/28; 005 10/15/24; 006 6/10/16. Estimates are unvalidated planning inputs, not delivery commitments.

## Technical approach

- Request microphone only after a clear user action/consent state; handle denial, unavailable devices and permission revocation.
- Use Web Audio/AudioWorklet only after browser support is confirmed. If resampling is needed, implement and test correct anti-alias filtering; do not assume changing a sample-rate label resamples audio.
- Send only audio bytes/framing required by the contract. Do not provide speaker identity as a trusted field.
- Stop capture on mute, stop, meeting exit and permission loss; release tracks, worklets and sockets. Reconnect behavior must align with server stream lifecycle and avoid retransmitting stale audio.
- Render Arabic/English mixed-script text correctly, preserve timestamps, announce updates accessibly and avoid appending every partial hypothesis as a new message.
- Client owns UX and capture diagnostics; backend owns authentication/identity decisions and AI owns recognition.

## Tools and technologies

Existing material references Angular 17+, Web Audio API, AudioWorklet and SignalR. Confirm versions and project patterns in implementation. Browser codecs, sample-rate conversion and media constraints remain implementation choices subject to testing.

## Deliverables

Capture component/service; audio-contract adapter; transcript component; permission/error/reconnect UX; browser support and accessibility verification.

## Acceptance criteria

Capture lifecycle obeys consent/mute/exit states; emitted audio conforms to the versioned contract; no client-provided identity is trusted; partial/final segments render in correct order for RTL Arabic and mixed language; resources are cleaned up; reconnect does not duplicate final records.

## Dependencies and open questions

Depends on supported browsers/devices, UX decision on opt-in, audio schema and reconnect semantics. Decide transcript retention in UI, copy/export behavior, speaker display for removed users, and treatment of multiple local audio devices.
