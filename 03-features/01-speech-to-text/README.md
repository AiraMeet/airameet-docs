# Speech-to-Text (STT)

This directory is the engineering source of truth for AiraMeet meeting transcription: Egyptian Arabic, English, Arabic-English code-switching, word/segment timestamps, authenticated participant attribution, and real-time meeting delivery. It records implementation intent and decisions; it does not claim that planned work has shipped or that unmeasured targets have been met.

## Reading order and phase dependencies

| Phase | Read | Outcome / dependency |
|---|---|---|
| 01. Planning & Design | [Overview](01-planning-design/overview.md), [Requirements](01-planning-design/requirements.md), [System architecture](01-planning-design/system-architecture.md) | Scope, traceable requirements, boundaries and proposed end-to-end design. Start here. |
| 02. Data Engineering | [Dataset pipeline](02-data-engineering/dataset-pipeline.md) | Reproducible, reviewed corpus and speaker-disjoint data splits; depends on approved data access and labeling policy. |
| 03. Model Development | [Model selection](03-model-development/model-selection.md), [Training and export](03-model-development/training-and-export.md), [Evaluation](03-model-development/evaluation.md) | Evidence-based model choice, reproducible training, and measured quality/runtime; depends on phase 02. |
| 04. System Integration | [Streaming contract](04-system-integration/streaming-contract.md), [Speaker attribution](04-system-integration/speaker-attribution.md), [Backend ownership](04-system-integration/backend-ownership.md), [Frontend ownership](04-system-integration/frontend-ownership.md) | Contracts between client, backend and AI; depends on phase 01 and iterates with phase 03. |
| 05. Quality & Delivery | [Quality assurance](05-quality-delivery/quality-assurance.md), [Operations](05-quality-delivery/operations.md) | Verification gates and deployment/operational readiness; depends on phases 03–04. |
| 06. Project Management | [Delivery plan](06-project-management/plan.md) | Workstream ownership, dependencies, risks, estimates and milestone planning across all phases. |

## Decision status

- **Confirmed project context:** AiraMeet needs Egyptian Arabic, English and Arabic-English code-switching transcription; speaker attribution, timestamps and real-time meeting integration are in scope.
- **Established design direction in source material:** attribute an isolated participant audio stream to the authenticated meeting identity; client-provided speaker IDs are untrusted. Treat this as an architectural constraint for the design, subject to the product's authentication and privacy policies.
- **Proposals / targets:** model family, audio format and chunking values, transport details, database schema, latency/accuracy goals, and deployment topology remain proposals or targets until measured, reviewed and approved. Numeric values in this documentation are not reported results.
- **TBD:** dataset inventory and consent/retention policy, model and license selection, evaluation thresholds, wire-level schemas, infrastructure sizing, and final milestone dates. Track decisions in the owning document and record approval/evidence before changing status.

## Document map

Existing files were reorganized as follows:

| Previous document | Current location |
|---|---|
| `feature-overview.md` | [01-planning-design/overview.md](01-planning-design/overview.md) |
| `requirements.md` | [01-planning-design/requirements.md](01-planning-design/requirements.md) |
| New synthesis of architecture content from overview, streaming and attribution docs | [01-planning-design/system-architecture.md](01-planning-design/system-architecture.md) |
| `dataset-preparation.md` | [02-data-engineering/dataset-pipeline.md](02-data-engineering/dataset-pipeline.md) |
| `model-selection.md` | [03-model-development/model-selection.md](03-model-development/model-selection.md) |
| `model-training.md` | [03-model-development/training-and-export.md](03-model-development/training-and-export.md) |
| `model-evaluation.md` | [03-model-development/evaluation.md](03-model-development/evaluation.md) |
| `real-time-transcription.md` | [04-system-integration/streaming-contract.md](04-system-integration/streaming-contract.md) |
| `speaker-identification.md` | [04-system-integration/speaker-attribution.md](04-system-integration/speaker-attribution.md) |
| `backend-tasks.md` | [04-system-integration/backend-ownership.md](04-system-integration/backend-ownership.md) |
| `frontend-tasks.md` | [04-system-integration/frontend-ownership.md](04-system-integration/frontend-ownership.md) |
| `testing.md` | [05-quality-delivery/quality-assurance.md](05-quality-delivery/quality-assurance.md) |
| `deployment.md` | [05-quality-delivery/operations.md](05-quality-delivery/operations.md) |
| `timeline.md` | [06-project-management/plan.md](06-project-management/plan.md) |

## Change control

Keep each requirement uniquely identified (`FR-STT-*`, `NFR-STT-*`) and reference those IDs from design, implementation and verification records. Mark statements **Confirmed**, **Proposed**, **Target**, or **TBD**. A proposal becomes confirmed only after the owning technical lead records the decision and its rationale; a target becomes achieved only when a reproducible measurement is attached. Update links here whenever documents move.
