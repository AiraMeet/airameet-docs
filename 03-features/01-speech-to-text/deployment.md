# Phase 9: Deployment & Operations (WBS)

## 1. Workstream Overview

This document specifies the deployment and operationalization tasks required to promote the Speech-to-Text and Speaker Attribution pipeline from staging into production. It addresses GPU resource provisioning, containerization, environment configuration, database migrations, observability, and rollback mechanisms.

All tasks follow the standardized task schema and three-point PERT estimation model ($E = \frac{O + 4M + P}{6}$).

---

## 2. Tasks Catalog: DevOps & Release Engineering

### `OPS-STT-001`: Docker Compose & GPU Containerization
- **Task ID**: `OPS-STT-001`
- **Task Name**: Build Production Docker Images and Compose Configuration
- **Description**: Package the FastAPI AI service with NVIDIA CUDA runtime (`nvidia/cuda:12.2.0-runtime-ubuntu22.04`), ASP.NET Core API with multi-stage SDK builds, and Angular client served via Nginx.
- **Priority**: Critical
- **Responsible Department**: DevOps & Infrastructure
- **Dependencies**: `BE-STT-004`, `FE-STT-003`
- **Implementation Steps**:
  1. Write `ai-services/Dockerfile` with `ctranslate2` and GPU support.
  2. Write `backend/AiraMeet.Api/Dockerfile` optimizing .NET 8 runtime images.
  3. Write `frontend/Dockerfile` building Angular static bundle into Nginx.
  4. Create `infrastructure/docker/docker-compose.prod.yml` with GPU pass-through configurations.
- **Time Estimates**:
  - Optimistic ($O$): 8h
  - Most Likely ($M$): 12h
  - Pessimistic ($P$): 20h
  - Expected ($E$): $\frac{8 + 4(12) + 20}{6} = 12.67\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Docker containers build successfully; FastAPI container detects and initializes NVIDIA GPU via `nvidia-smi`.
- **Deliverable**: `Dockerfile` manifests and `docker-compose.prod.yml`.
- **Status**: `To Do`

---

### `OPS-STT-002`: GPU Warm-Up, Memory Management & Health Probes
- **Task ID**: `OPS-STT-002`
- **Task Name**: Implement GPU Startup Warm-Up and Kubernetes/Docker Health Probes
- **Description**: Add application lifespan event hooks to load models into GPU memory and run synthetic forward passes during startup, preventing cold-start lag for users. Implement liveness and readiness health checks.
- **Priority**: High
- **Responsible Department**: AI Engineering & DevOps
- **Dependencies**: `OPS-STT-001`
- **Implementation Steps**:
  1. Add FastAPI startup hook to load CTranslate2 model and run 2.0s synthetic silence audio.
  2. Implement `/health/live` and `/health/ready` endpoints verifying CUDA device readiness.
  3. Add ASP.NET Core health check querying SQL Server, Redis, and the AI health endpoint.
- **Time Estimates**:
  - Optimistic ($O$): 6h
  - Most Likely ($M$): 8h
  - Pessimistic ($P$): 14h
  - Expected ($E$): $\frac{6 + 4(8) + 14}{6} = 8.67\text{h}$
  - Risk Buffer: 1.5h
- **Acceptance Criteria**: Services reject traffic until GPU warm-up completes; health probes respond `200 OK` when ready.
- **Deliverable**: Lifespan startup hooks and `/health` controllers.
- **Status**: `To Do`

---

### `OPS-STT-003`: Database Migration CI/CD Integration
- **Task ID**: `OPS-STT-003`
- **Task Name**: Automate SQL Server Entity Framework Migrations in CI/CD
- **Description**: Establish an automated deployment step executing EF Core migrations against the target database before rolling out new application backend containers.
- **Priority**: High
- **Responsible Department**: DevOps & Backend
- **Dependencies**: `BE-STT-001`
- **Implementation Steps**:
  1. Configure `dotnet ef migrations bundle` script in GitHub Actions.
  2. Add pre-deployment database migration job with automated rollback on failure.
  3. Verify idempotent execution against existing schema.
- **Time Estimates**:
  - Optimistic ($O$): 4h
  - Most Likely ($M$): 6h
  - Pessimistic ($P$): 10h
  - Expected ($E$): $\frac{4 + 4(6) + 10}{6} = 6.33\text{h}$
  - Risk Buffer: 1h
- **Acceptance Criteria**: Database schema migrations execute seamlessly in CI pipeline without manual intervention.
- **Deliverable**: GitHub Actions migration workflow and migration bundle script.
- **Status**: `To Do`

---

### `OPS-STT-004`: Telemetry, Observability & Prometheus Dashboards
- **Task ID**: `OPS-STT-004`
- **Task Name**: Configure OpenTelemetry, Prometheus Metrics, and Grafana Dashboard
- **Description**: Instrument ASP.NET Core and FastAPI services with OpenTelemetry metrics: track inference latency, active audio streams, WebSocket connection drops, and GPU VRAM saturation.
- **Priority**: Medium
- **Responsible Department**: DevOps & Platform Engineering
- **Dependencies**: `OPS-STT-001`
- **Implementation Steps**:
  1. Add `prometheus-fastapi-instrumentator` to AI service.
  2. Add OpenTelemetry metrics exporter in ASP.NET Core `Program.cs`.
  3. Provision Grafana dashboard displaying real-time STT latency percentiles (p50, p95, p99).
- **Time Estimates**:
  - Optimistic ($O$): 8h
  - Most Likely ($M$): 12h
  - Pessimistic ($P$): 18h
  - Expected ($E$): $\frac{8 + 4(12) + 18}{6} = 12.33\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Grafana dashboard displays live inference latency, active room counts, and GPU memory metrics.
- **Deliverable**: Prometheus configuration and Grafana dashboard JSON manifest.
- **Status**: `To Do`

---

### `OPS-STT-005`: Production Deployment Verification & Rollback Runbook
- **Task ID**: `OPS-STT-005`
- **Task Name**: Execute Staging Smoke Tests and Publish Rollback Runbook
- **Description**: Conduct full smoke testing on the staging environment, document incident response runbooks, and test hot-swap rollback of AI model weights without restarting core API services.
- **Priority**: Critical
- **Responsible Department**: DevOps & All Leads
- **Dependencies**: `OPS-STT-001`, `OPS-STT-002`, `OPS-STT-003`, `OPS-STT-004`
- **Implementation Steps**:
  1. Deploy release to staging cluster.
  2. Execute production verification checklist (smoke test live meeting with audio).
  3. Test model rollback procedure by pointing `MODEL_PATH` to previous baseline.
  4. Publish formal release runbook.
- **Time Estimates**:
  - Optimistic ($O$): 6h
  - Most Likely ($M$): 10h
  - Pessimistic ($P$): 16h
  - Expected ($E$): $\frac{6 + 4(10) + 16}{6} = 10.33\text{h}$
  - Risk Buffer: 2h
- **Acceptance Criteria**: Rollback completes in under 60 seconds; production verification checklist signed off.
- **Deliverable**: `docs/06-deployment/operational-runbook.md` and release sign-off.
- **Status**: `To Do`
