# Phase 7: Frontend Development Tasks (WBS)

## 1. Workstream Overview

This document specifies the complete engineering Work Breakdown Structure (WBS) for the Frontend workstream, implemented in **Angular 17+ (TypeScript)** using Standalone Components, Reactive Signals, and Tailwind CSS.

All tasks follow the project's standardized task schema and three-point PERT estimation model ($E = \frac{O + 4M + P}{6}$).

---

## 2. Tasks Catalog: Angular Client

### `FE-STT-001`: Authentication Views & Route Guards
- **Task ID**: `FE-STT-001`
- **Task Name**: Build Registration, Login Views, Form Validation, and Auth Guards
- **Description**: Implement responsive registration and login forms with reactive validation, JWT token storage, refresh interceptor, and `canActivate` authentication route guards.
- **Priority**: Critical
- **Responsible Department**: Frontend Engineering (Angular)
- **Dependencies**: Existing ASP.NET Core `/api/auth` endpoints
- **Implementation Steps**:
  1. Scaffold `LoginComponent` and `RegisterComponent` with reactive forms.
  2. Implement `AuthService` managing JWT tokens and refresh token rotation.
  3. Create HTTP interceptor injecting `Authorization: Bearer <token>` on API requests.
  4. Create `authGuard` preventing unauthenticated access to `/dashboard` and `/meeting`.
- **Time Estimates**:
  - Optimistic ($O$): 10h
  - Most Likely ($M$): 14h
  - Pessimistic ($P$): 22h
  - Expected ($E$): $\frac{10 + 4(14) + 22}{6} = 14.67\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Users can register, log in, view validation errors, access protected routes, and automatically refresh expired tokens.
- **Deliverable**: Angular auth module, components, services, and route guards.
- **Status**: `To Do`

---

### `FE-STT-002`: Dashboard, Meeting Creation & Join Modals
- **Task ID**: `FE-STT-002`
- **Task Name**: Implement User Dashboard, Create Meeting Modal, and Join Interface
- **Description**: Build the authenticated home dashboard displaying user profile details, recent meeting history, a "New Meeting" creation modal, and a "Join via Code" interface.
- **Priority**: High
- **Responsible Department**: Frontend Engineering (Angular)
- **Dependencies**: `FE-STT-001`, `BE-STT-002`
- **Implementation Steps**:
  1. Scaffold `DashboardComponent` with user profile header.
  2. Implement `CreateMeetingModal` with title, agenda, and scheduling fields.
  3. Implement `JoinMeetingComponent` validating 10-character room codes.
  4. Implement `RecentMeetingsComponent` listing past sessions with transcript links.
- **Time Estimates**:
  - Optimistic ($O$): 8h
  - Most Likely ($M$): 12h
  - Pessimistic ($P$): 18h
  - Expected ($E$): $\frac{8 + 4(12) + 18}{6} = 12.33\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: User creates a room, receives a shareable meeting URL/code, joins existing rooms, and navigates directly to the room view.
- **Deliverable**: `DashboardComponent`, `MeetingModalComponent`, and `MeetingService`.
- **Status**: `To Do`

---

### `FE-STT-003`: Meeting Room Layout, Grid & Participant Controls
- **Task ID**: `FE-STT-003`
- **Task Name**: Implement Meeting Room Container, Video Grid, and Controls Bar
- **Description**: Build the main meeting room container including video tile grid, bottom control bar (Mic, Camera, Screen Share, Leave), participant list drawer, and responsive layout.
- **Priority**: Critical
- **Responsible Department**: Frontend Engineering (Angular)
- **Dependencies**: `FE-STT-002`
- **Implementation Steps**:
  1. Scaffold `MeetingRoomComponent` with CSS Grid layout.
  2. Implement `MeetingControlsBarComponent` with reactive mute/unmute buttons.
  3. Implement `ParticipantListDrawerComponent` showing active speakers and roles.
  4. Add responsive handling for desktop, tablet, and mobile orientations.
- **Time Estimates**:
  - Optimistic ($O$): 12h
  - Most Likely ($M$): 16h
  - Pessimistic ($P$): 26h
  - Expected ($E$): $\frac{12 + 4(16) + 26}{6} = 17.00\text{h}$
  - Risk Buffer: 3h
- **Acceptance Criteria**: Meeting room renders video placeholders, participant tiles, functional mute/camera toggle controls, and side drawer.
- **Deliverable**: `MeetingRoomComponent`, controls bar, and participant drawer.
- **Status**: `To Do`

---

### `FE-STT-004`: AudioWorklet Microphone Capture & 16 kHz Ingestion
- **Task ID**: `FE-STT-004`
- **Task Name**: Implement Web Audio API AudioWorklet 16 kHz Mono Ingestion
- **Description**: Develop an `AudioWorkletProcessor` to tap the microphone `MediaStream`, downsample hardware audio to 16 kHz mono signed 16-bit PCM, and stream binary chunks every 250ms over WebSocket.
- **Priority**: Critical
- **Responsible Department**: Frontend Engineering (Angular)
- **Dependencies**: `FE-STT-003`, `BE-STT-003`
- **Implementation Steps**:
  1. Write `audio-stream-processor.js` registered in `AudioWorklet`.
  2. Implement `AudioCaptureService` in Angular requesting `navigator.mediaDevices.getUserMedia`.
  3. Establish authenticated WebSocket connection to `/ws/meetings/{id}/audio-stream`.
  4. Pipe 250ms `Int16Array` buffers from worklet port directly to the WebSocket.
  5. Cease transmission immediately when user toggles microphone mute.
- **Time Estimates**:
  - Optimistic ($O$): 12h
  - Most Likely ($M$): 18h
  - Pessimistic ($P$): 28h
  - Expected ($E$): $\frac{12 + 4(18) + 28}{6} = 18.67\text{h}$
  - Risk Buffer: 3h
- **Acceptance Criteria**: AudioWorklet runs without blocking UI; transmits regular 250ms binary frames; halts transmission on mute; verified 16 kHz mono.
- **Deliverable**: `AudioCaptureService.ts` and `audio-stream-processor.js`.
- **Status**: `To Do`

---

### `FE-STT-005`: Real-Time Live Transcript Component & Speaker Badges
- **Task ID**: `FE-STT-005`
- **Task Name**: Build Live Transcript Drawer with Partial/Final Text & Speaker Attribution
- **Description**: Implement the real-time transcript drawer in Angular that connects to the SignalR hub, dynamically renders partial and final speech chunks, attributes text to speakers, and auto-scrolls.
- **Priority**: Critical
- **Responsible Department**: Frontend Engineering (Angular)
- **Dependencies**: `FE-STT-003`, `BE-STT-006`
- **Implementation Steps**:
  1. Scaffold `LiveTranscriptDrawerComponent`.
  2. Implement `TranscriptSignalRService` listening to `ReceiveTranscriptSegment`.
  3. Manage transcript state with Angular Signals:
     - Render final transcripts with speaker name, timestamp, and avatar.
     - Render currently active partial transcript in italics beneath the speaker's block.
  4. Implement auto-scroll with user scroll lock override.
- **Time Estimates**:
  - Optimistic ($O$): 10h
  - Most Likely ($M$): 15h
  - Pessimistic ($P$): 24h
  - Expected ($E$): $\frac{10 + 4(15) + 24}{6} = 15.67\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Transcripts render with $< 200\text{ms}$ UI delay; partial text updates smoothly; finalized text locks in place with speaker avatar and timestamp.
- **Deliverable**: `LiveTranscriptDrawerComponent` and `TranscriptSignalRService`.
- **Status**: `To Do`

---

### `FE-STT-006`: Network Resilience, Loading, Error & Disconnected States
- **Task ID**: `FE-STT-006`
- **Task Name**: Implement Connection Recovery, Reconnection Banners, and Error States
- **Description**: Add visual banners and state handlers for network drops, WebSocket disconnections, microphone permission denials, and server reconnection retries.
- **Priority**: Medium
- **Responsible Department**: Frontend Engineering (Angular)
- **Dependencies**: `FE-STT-004`, `FE-STT-005`
- **Implementation Steps**:
  1. Implement exponential backoff reconnect handler in `TranscriptSignalRService` and `AudioCaptureService`.
  2. Create `ConnectionStatusBannerComponent` (Connected, Reconnecting, Disconnected).
  3. Handle microphone permission rejection with actionable instructional dialogs.
- **Time Estimates**:
  - Optimistic ($O$): 6h
  - Most Likely ($M$): 10h
  - Pessimistic ($P$): 16h
  - Expected ($E$): $\frac{6 + 4(10) + 16}{6} = 10.33\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Client automatically recovers after temporary socket disconnect without page refresh; visual feedback clearly displayed.
- **Deliverable**: `ConnectionStatusBannerComponent.ts` and error interceptors.
- **Status**: `To Do`
