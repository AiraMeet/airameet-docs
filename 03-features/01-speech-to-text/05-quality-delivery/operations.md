# 05. Quality & Delivery — Operations, Deployment & Runbook

## 1. Overview & Operational Scope

This document details the operational deployment architecture for the AiraMeet Speech-to-Text and Diarization services, including folder structure specifications, multi-stage GPU containerization, Prometheus/OpenTelemetry telemetry, health check probes, and disaster recovery procedures.

---

## 2. Complete AI Service Folder Structure

The AI service repository is strictly organized to decouple training from inference runtime, and to isolate models and datasets from source code:

```text
ai-service/
├── configs/                                # Declarative YAML configurations
│   ├── dataset_config.yaml                 # Paths, audio criteria, normalization rules
│   ├── training_config.yaml                # Hyperparameters, batch sizes, optimizer, LoRA
│   ├── diarization_config.yaml             # PyAnnote parameters, clustering thresholds
│   └── inference_config.yaml               # Serving parameters, VAD thresholds, batch timeouts
├── data/                                   # Local storage for datasets (gitignored)
│   ├── raw/                                # Original audio files and CSV metadata
│   ├── processed/                          # Resampled 16 kHz Mono WAV files
│   └── manifests/                          # train.jsonl, val.jsonl, test.jsonl
├── models/                                 # Model weights and artifacts (gitignored)
│   ├── pretrained/                         # Downloaded baseline Hugging Face models
│   ├── checkpoints/                        # Active rolling training step checkpoints
│   ├── best_model/                         # Validated best ASR checkpoint
│   └── exported/                           # CTranslate2 INT8 quantized production models
├── src/                                    # Reusable application and library code
│   ├── common/                             # Shared utilities (logging, audio_utils)
│   ├── dataset/                            # Audio validator, text normalizer, manifest builder
│   ├── asr/                                # Dataset loader, scoring metrics, CTranslate2 engine
│   ├── diarization/                        # PyAnnote engine, timestamp aligner
│   └── pipeline/                           # Silero VAD, sliding buffer, orchestrator
├── service/                                # Serving & deployment interfaces
│   ├── api/                                # REST API endpoints (routes_health, routes_offline)
│   ├── grpc/                               # gRPC real-time streaming interfaces & proto stubs
│   └── main.py                             # Service entry point (FastAPI + gRPC runners)
├── scripts/                                # Executable CLI scripts (not imported as modules)
│   ├── run_inspection.py                   # Run dataset inspection and quality validation
│   ├── run_dataset_prep.py                 # Execute full dataset cleaning and splitting
│   ├── run_baseline_eval.py                # Evaluate zero-shot baseline WER/CER
│   ├── run_training.py                     # Launch ASR fine-tuning
│   ├── run_evaluation.py                   # Benchmark fine-tuned model against test set
│   ├── run_export.py                       # Export best model to CTranslate2 INT8
│   └── run_diarization_eval.py             # Benchmark PyAnnote on RTTM validation set
├── tests/                                  # Automated test suite (unit, integration, fixtures)
├── Dockerfile                              # Multi-stage GPU Dockerfile
├── requirements.txt                        # Pinned production dependencies
├── requirements-dev.txt                    # Development, testing, and training dependencies
└── README.md                               # Service documentation and execution instructions
```

---

## 3. Production Containerization (Multi-Stage GPU Dockerfile)

```dockerfile
# Stage 1: Build & Dependency Resolution
FROM nvidia/cuda:12.2.2-runtime-ubuntu22.04 AS builder

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3.11 \
    python3.11-dev \
    python3-pip \
    python3.11-venv \
    ffmpeg \
    libsndfile1 \
    git \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

RUN python3.11 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --upgrade pip setuptools wheel && \
    pip install -r requirements.txt

# Stage 2: Minimal Production Runtime
FROM nvidia/cuda:12.2.2-runtime-ubuntu22.04 AS runner

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3.11 \
    ffmpeg \
    libsndfile1 \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /opt/venv /opt/venv

WORKDIR /app
COPY src/ /app/src/
COPY service/ /app/service/
COPY configs/inference_config.yaml /app/configs/inference_config.yaml
COPY models/exported/whisper_medium_ct2/ /app/models/exported/whisper_medium_ct2/

EXPOSE 8000 50051

HEALTHCHECK --interval=15s --timeout=5s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8000/health/liveness || exit 1

ENTRYPOINT ["python3.11", "-m", "service.main"]
```

---

## 4. Telemetry, Metrics & Observability

The AI service exposes Prometheus metrics at `/metrics` (port 8000):

| Metric Name | Type | Description | Target SLA / Alert Threshold |
| :--- | :--- | :--- | :--- |
| `airameet_ai_inference_duration_seconds` | Histogram | Forward pass duration of Whisper INT8. | p95 $\le 0.25\text{s}$; Alert if $> 0.50\text{s}$. |
| `airameet_ai_real_time_factor` | Gauge | Real-Time Factor (RTF) across streams. | Target $\le 0.15$; Alert if $> 0.25$. |
| `airameet_ai_active_streams_count` | Gauge | Number of active concurrent audio streams. | Max capacity 16 per GPU worker. |
| `airameet_ai_vad_speech_ratio` | Histogram | Percentage of incoming audio marked as speech. | Monitored for VAD drift. |
| `airameet_ai_gpu_memory_used_bytes` | Gauge | NVIDIA VRAM utilization. | Alert if $> 85\%$ of total VRAM. |

---

## 5. Health Checks & Probes

1. **Liveness Probe (`GET /health/liveness`)**:
   - Returns `200 OK` if the Python event loop is responsive.
   - Used by Kubernetes/Docker to restart deadlocked workers.
2. **Readiness Probe (`GET /health/readiness`)**:
   - Returns `200 OK` only when:
     - Whisper CTranslate2 model is loaded into GPU memory.
     - Silero VAD TorchScript engine is initialized.
     - PyAnnote diarization pipeline is loaded.
     - gRPC port 50051 is accepting connections.

---

## 6. Disaster Recovery & Rollback Runbook

### Scenario: High Inference Latency or VRAM Spike
1. **Detection**: Prometheus alert fires: `airameet_ai_real_time_factor > 0.25` for 3 consecutive minutes.
2. **Immediate Remediation**:
   - Check GPU load via `nvidia-smi`.
   - Scale horizontally by adding a second AI worker container: `docker compose up -d --scale ai-service=2`.
   - ASP.NET Core gRPC load balancer automatically distributes incoming streams across both workers.
3. **Rollback Strategy**:
   - If a new model version (`v1.1`) exhibits elevated WER or memory leaks, update the environment variable `AI_MODEL_TAG=v1.0-stable` in `docker-compose.yml`.
   - Execute rolling restart: `docker compose up -d --no-deps ai-service`.
   - ASP.NET Core retries in-flight frames and seamlessly reconnects to the stable container.
