# Project Overview: AiraMeet

## 1. Executive Summary

**AiraMeet** is an intelligent, enterprise-grade video conferencing platform designed to bridge the gap between synchronous virtual collaboration and asynchronous knowledge execution. While conventional video conferencing tools (Zoom, Google Meet, Microsoft Teams) primarily focus on transmitting audio and video, AiraMeet treats meeting dialogue as an active stream of knowledge. 

By integrating deep learning speech recognition and natural language understanding directly into the meeting lifecycle, AiraMeet automatically produces timestamped, speaker-attributed transcripts, extracts decisions and action items, tracks open issues, and creates searchable meeting knowledge graphs.

---

## 2. Business Objectives & Value Proposition

Virtual meetings frequently suffer from several organizational failure modes:
1. **Context Loss**: Critical architectural decisions, agreements, and rationale shared verbally are lost unless manually documented.
2. **Ambiguous Accountability**: Tasks and deadlines mentioned during discussions fail to translate into actionable project tracking entries.
3. **Information Silos**: Team members absent from meetings must read through disjointed notes or spend hours reviewing full video recordings.
4. **Multilingual Inefficiencies**: Cross-border teams struggle with regional accents and multilingual dialogues without reliable real-time transcription.

### Key Value Drivers
- **Verifiable Decision Tracking**: Every extracted decision links directly back to the exact audio segment and timestamp where it was formulated.
- **Automated Task Governance**: Action items, assignees, and deadlines are extracted in real-time, ready for export or sync with issue trackers.
- **Accurate Speaker Attribution**: Using authenticated session routing, spoken words are deterministically attributed to logged-in meeting participants without requiring expensive acoustic diarization models.
- **High-Performance Privacy**: Core AI workloads can be hosted on private or dedicated infrastructure, safeguarding confidential organizational IP.

---

## 3. Product Scope & Minimum Viable Product (MVP)

AiraMeet is developed iteratively. The initial scope focuses on building a rock-solid, production-grade foundation followed by incremental AI intelligence layers.

### Phase 1: MVP Core Platform (Current Focus)
- **Identity & Access Management**: Secure user registration, authentication, JWT token issuance with rotating refresh tokens, and user profiles.
- **Meeting Room Management**: Room creation, unique meeting URLs/IDs, host permissions, participant admissions, and participant lists.
- **Real-Time Communication**: Multi-party WebRTC audio and video conferencing, mute/unmute states, camera toggling, and screen sharing.
- **Feature 01 - Speech-to-Text & Speaker Attribution**:
  - Live audio ingestion from individual participant streams.
  - Deep learning speech-to-text inference with low latency (< 3 seconds).
  - Speaker identification derived from authenticated session tokens.
  - Live transcript stream displayed dynamically in the Angular web client.
  - Persistent storage of final meeting transcripts in SQL Server.

### Phase 2: Post-MVP Extensions (Planned)
- Real-time meeting agent and discussion flow analyzer.
- Decision extraction and action item follow-up engine.
- Cross-meeting semantic search using vector embeddings.
- Meeting knowledge graph construction.
- AI meeting copilot with factual evidence retrieval.

---

## 4. Key Stakeholders & Personas

| Persona | Role | Primary Needs |
| :--- | :--- | :--- |
| **Engineering Lead / Architect** | Technical Leader | High-fidelity technical discussions, automatic tracking of architectural decisions, zero data leakage. |
| **Product Manager / Scrum Master** | Facilitator | Automatic extraction of sprint tasks, clear accountability, reduced overhead in writing meeting summaries. |
| **Remote Developer / Student** | Meeting Participant | Real-time transcript display for clarity, speaker labels, searchable history for review. |
| **System Administrator / DevOps** | Infrastructure Owner | Scalable containerized deployment, clear GPU resource limits, observable logs and metrics. |

---

## 5. Architectural & Governance Principles

1. **Clean Separation of Concerns**: 
   - Business state and authorization are governed strictly by the ASP.NET Core backend.
   - User presentation and WebRTC client orchestration are handled by the Angular frontend.
   - Deep learning inference is isolated in a scalable Python FastAPI service with dedicated GPU access.
2. **Deterministic Speaker Attribution**: 
   - Never trust client-reported user IDs in raw audio frames. 
   - The backend server authenticates the WebSocket/WebRTC connection and binds the audio pipeline directly to the authenticated user ID.
3. **Data Integrity & Privacy**:
   - Audio streams must be processed transiently in memory buffers; raw audio is not stored permanently unless explicitly consented to by meeting participants for recording.
4. **Resilience & Graceful Degradation**:
   - If AI transcription experiences heavy load or GPU failure, core audio/video conferencing must remain completely operational.

---

## 6. Document References & Cross-Links

- Technical Stack Overview: [technology-stack.md](technology-stack.md)
- System Architecture Deep Dive: [system-architecture.md](system-architecture.md)
- Master Roadmap: [master-roadmap.md](../02-planning/master-roadmap.md)
- Feature 01 - Speech-to-Text: [01-speech-to-text/README.md](../03-features/01-speech-to-text/README.md)
