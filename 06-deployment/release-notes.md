# Release Notes & Versioning Governance

## 1. Release Governance & Semantic Versioning

AiraMeet follows strict [Semantic Versioning (SemVer 2.0.0)](https://semver.org/):
$$\text{MAJOR}.\text{MINOR}.\text{PATCH}$$
- **MAJOR**: Incompatible API changes, breaking database schema shifts, or major architectural overhauls.
- **MINOR**: Backward-compatible new capabilities (e.g., adding Speech-to-Text, introducing new export formats).
- **PATCH**: Backward-compatible bug fixes, performance optimizations, and security patches.

---

## 2. Release History & Milestones

### Version 0.2.0 (Target: Week 6) - *Speech-to-Text & Speaker Attribution Release*
**Status**: Planned / In Development  
**Target Release Date**: 2027-01-08

#### New Features
- **Deep Learning Speech-to-Text**: Real-time transcription engine powered by fine-tuned speech recognition models (Whisper/Wav2Vec2/NeMo).
- **Session-Bound Speaker Attribution**: Spoken utterances automatically attributed to logged-in meeting participants via authenticated connection binding.
- **Dual-Track Audio Streaming**: Web Audio API `AudioWorklet` capturing 16 kHz linear PCM without impacting interactive WebRTC video/audio calls.
- **Sliding-Window Transcription Engine**: 1.5-second chunking with 300ms overlap and Silero Voice Activity Detection.
- **Boundary Deduplication**: Algorithmic longest common substring pruning to eliminate duplicated words across streaming chunk boundaries.
- **Live Transcript UI**: Angular meeting room transcript drawer with real-time scrolling, partial (interim) text, and final text states.
- **Transcript Persistence**: Relational storage of finalized transcript segments in SQL Server with millisecond timestamps and confidence scores.

#### Architecture & Infrastructure Changes
- Added `ai-services` container running FastAPI, PyTorch, and CTranslate2 with GPU pass-through.
- Added database tables: `Meetings`, `MeetingParticipants`, and `TranscriptSegments`.
- Integrated Redis pub/sub backplane for horizontal SignalR streaming scaling.

---

### Version 0.1.0 - *Foundation & Authentication Release*
**Status**: Completed  
**Release Date**: 2026-10-30

#### Features Delivered
- ASP.NET Core Web API with Clean Architecture structure (`AiraMeet.Api`, `AiraMeet.Application`, `AiraMeet.Domain`, `AiraMeet.Infrastructure`).
- ASP.NET Core Identity integration with custom `ApplicationUser` using `Guid` keys.
- JWT authentication issuance and validation with 15-minute access token lifespan.
- Cryptographically secure refresh token storage with rotation and revocation.
- Global exception handling middleware and RFC 7807 Problem Details responses.
- Initial Angular frontend asset preparation and workspace layout.

---

## 3. Pre-Release Checklist & Migration Procedures

Before promoting any release to production, the release engineering team must execute the following checklist:
1. **Database Schema Migration**:
   ```bash
   dotnet ef database update --project backend/AiraMeet.Infrastructure
   ```
2. **GPU Inference Readiness Verification**:
   - Check CUDA driver compatibility: `nvidia-smi`.
   - Run warm-up test: `curl -f http://ai-service:8000/health/ready`.
3. **Automated Test Pass Verification**:
   - Verify unit test suite passing $\ge 85\%$ coverage.
   - Run audio streaming load test (`TC-STREAM-002`).
4. **Environment Secrets Audit**:
   - Ensure production JWT secrets and database connection strings are injected via Azure Key Vault / AWS Secrets Manager.
