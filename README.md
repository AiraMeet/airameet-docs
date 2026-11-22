# AiraMeet

**Turn live conversations into searchable transcripts, decisions, and follow-up actions.**

AiraMeet is an AI-powered video conferencing platform. This repository is its documentation hub, covering product scope, architecture, planning, feature specifications, testing, and deployment.

## Description

AiraMeet brings real-time meetings and AI-assisted understanding together. The platform is designed around a web client, a business API, a real-time media pipeline, and AI services for transcription and other meeting intelligence capabilities.

## Documentation

Start with the [project overview](01-project-overview/project-overview.md) for the product vision, goals, and MVP scope.

| If you want to learn about... | Read... |
| --- | --- |
| System components and architecture | [System architecture](01-project-overview/system-architecture.md) |
| Technologies in use and under consideration | [Technology stack](01-project-overview/technology-stack.md) |
| Milestones and delivery plan | [Master roadmap](02-planning/master-roadmap.md) · [Project timeline](02-planning/project-timeline.md) |
| Team responsibilities | [Team assignments](02-planning/team-assignments.md) |
| Speech-to-text feature | [Feature guide](03-features/01-speech-to-text/README.md) |
| API and WebSocket contracts | [API documentation](04-architecture/api-documentation.md) |
| Audio streaming | [Audio streaming design](04-architecture/audio-streaming.md) |
| Database design | [Database schema](04-architecture/database-schema.md) |
| Quality strategy and test cases | [Test strategy](05-testing/test-strategy.md) · [Test cases](05-testing/test-cases.md) |
| Deployment and operations | [Deployment guide](06-deployment/deployment-guide.md) · [Release notes](06-deployment/release-notes.md) |

## Architecture

The system separates the browser experience, business logic, media forwarding, and AI inference. The web client communicates with the API over HTTPS and WebSockets, while the media gateway carries WebRTC streams. Backend and AI services exchange meeting data and audio for transcription.

```mermaid
flowchart LR
    Client[Angular web client] <-->|HTTPS / WebSockets| API[ASP.NET Core API]
    Client <-->|WebRTC| Media[Media gateway / SFU]
    API <--> SQL[(SQL Server)]
    API <--> Redis[(Redis)]
    Media -->|Audio frames| AI[FastAPI AI services]
    API <-->|Internal API / gRPC| AI
    AI --> STT[Speech-to-text workers]
```

See the [system architecture](01-project-overview/system-architecture.md) and [technology stack](01-project-overview/technology-stack.md) for details and implementation decisions.

## Project structure

- [Project overview](01-project-overview/project-overview.md) — product direction and system context
- [Planning](02-planning/) — roadmap, timeline, and team assignments
- [Features](03-features/) — feature requirements, designs, tasks, tests, and timelines
- [Architecture](04-architecture/) — API, streaming, and data design
- [Testing](05-testing/) — quality strategy and test cases
- [Deployment](06-deployment/) — deployment guidance and release notes

Feature folders contain the requirements, design, tasks, testing approach, and delivery timeline for each feature. Use the [feature template](03-features/feature-template/README.md) when documenting a new one.

## Documentation standards

- Link requirements to designs, task IDs, and test cases.
- Estimate work with the three-point PERT formula: `E = (O + 4M + P) / 6`, including risk buffers.
- Mark unverified details as `[Assumption]` or `[To Be Decided - TBD]`.
- Keep examples and workflows reproducible and syntactically valid.
