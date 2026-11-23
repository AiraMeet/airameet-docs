# 04. Model Development — Evaluation & Benchmarking

## 1. Overview & Verification Strategy

Model evaluation provides empirical, reproducible proof of transcription accuracy and latency performance across dialects and code-switching scenarios before any service is released to staging or production.

---

## 2. Quantitative Acceptance Gates

| Metric | Baseline Target (Whisper-Medium) | Fine-Tuned Acceptance Gate | Test Verification Set |
| :--- | :---: | :---: | :--- |
| **Overall WER** | 27.8% | **$\le$ 18.0%** | Full held-out test split. |
| **Egyptian Arabic WER** | 29.5% | **$\le$ 19.5%** | Pure Arabic utterances. |
| **English WER** | 11.8% | **$\le$ 9.5%** | Pure English utterances. |
| **Code-Switching WER** | 34.2% | **$\le$ 22.0%** | Mixed Arabic-English utterances. |
| **Relative WER Improvement** | N/A | **$\ge$ 15.0%** | Over baseline on identical test split. |
| **Character Error Rate (CER)**| 14.5% | **$\le$ 9.0%** | Full held-out test split. |
| **Real-Time Factor (RTF)** | 0.28 (PyTorch FP16) | **$\le$ 0.15** (CTranslate2 INT8) | RTX 4090 / A10G GPU. |
| **GPU VRAM Usage** | 4.8 GB | **$\le$ 2.5 GB** | Peak memory during 10 concurrent streams. |
| **Diarization Error Rate (DER)**| N/A | **$\le$ 18.0%** | Egyptian Arabic RTTM validation set. |

---

## 3. Engineering Tasks & Subtasks Breakdown

---

### Task `AI-EVAL-01`: Comprehensive Model Evaluation & Multi-Dialect Benchmark

| Attribute | Specification |
| :--- | :--- |
| **Task ID** | `AI-EVAL-01` |
| **Task Name** | Comprehensive Model Evaluation & Multi-Dialect Benchmark |
| **Phase** | AI Phase 4: ASR Evaluation & Validation |
| **Priority / Owner** | `Critical` / AI Engineering Team |
| **Dependencies** | `AI-ASR-03` (Exported Model) |
| **Checkpoint** | **Yes** — Gate M3 Evaluation Sign-off Milestone (`reports/fine_tuned_scorecard.json`) |

#### Objective & Scope
Perform an exhaustive, reproducible benchmark of the exported CTranslate2 INT8 model across the isolated held-out test split (`test.jsonl`). Calculate Word Error Rate (WER) and Character Error Rate (CER) stratified across pure Arabic, pure English, and code-switched audio. Measure inference latency, RTF, peak VRAM, and generate an error analysis taxonomy report.

#### Subtasks Breakdown

| Subtask ID | Subtask Name | Actionable Steps & Implementation Scope | Output & Acceptance Criteria |
| :--- | :--- | :--- | :--- |
| `AI-EVAL-01.1` | Benchmark Script Implementation | Implement `scripts/run_evaluation.py` configured for CTranslate2 INT8 execution, reading records from `data/manifests/test.jsonl`. | Working benchmark script executing batched INT8 Whisper inference on GPU. |
| `AI-EVAL-01.2` | Batch Inference Execution | Execute batched model inference on all test audio files, logging per-utterance recognized text and inference duration in milliseconds. | Raw transcript hypotheses and execution timing logs recorded per sample. |
| `AI-EVAL-01.3` | Stratified Metric Calculation | Normalize references and predictions via `src/dataset/text_normalizer.py`. Calculate overall WER, CER, pure Egyptian Arabic WER, pure English WER, and code-switching WER using `jiwer`. | Accurate mathematical calculation of stratified error rates across language categories. |
| `AI-EVAL-01.4` | Compute Profiling & Error Analysis | Measure peak VRAM allocation via `pynvml`, calculate average RTF, categorize common error patterns in `reports/error_analysis_report.md`, and persist machine-readable results to `reports/fine_tuned_scorecard.json`. | Detailed error taxonomy report and machine-readable JSON scorecard. |

#### Expected Deliverables & Acceptance Criteria
- **Evaluation Tooling**: `scripts/run_evaluation.py`.
- **Scorecard Report**: `reports/fine_tuned_scorecard.json`.
- **Error Analysis**: `reports/error_analysis_report.md`.
- **Acceptance Gate**:
  - Overall WER $\le 18.0\%$ on held-out test split.
  - Code-Switching WER $\le 22.0\%$.
  - Relative WER reduction $\ge 15.0\%$ over baseline.
  - Real-Time Factor (RTF) $\le 0.15$.

#### Checkpoint Specification
- **Checkpoint Required**: **Yes** (Gate M3 Milestone).
- **Checkpoint Type**: Benchmark Verification Record (`asr-eval-v1.0`).
- **Contents**: Detailed JSON scorecard, error analysis markdown, and per-utterance scoring ledger.
- **Retention**: Permanent.

---

### Task `AI-EVAL-02`: Diarization Accuracy Validation (DER & Overlap Assessment)

| Attribute | Specification |
| :--- | :--- |
| **Task ID** | `AI-EVAL-02` |
| **Task Name** | Diarization Accuracy Validation (DER & Overlap Assessment) |
| **Phase** | AI Phase 5: Diarization Validation |
| **Priority / Owner** | `High` / AI Engineering Team |
| **Dependencies** | `AI-DIAR-01` (Diarization Pipeline), `AI-DIAR-02` (Alignment) |
| **Checkpoint** | **Yes** — Gate M5 Diarization Acceptance Gate |

#### Objective & Scope
Benchmark the PyAnnote diarization pipeline against ground-truth RTTM annotations from 5 multi-speaker Egyptian Arabic meetings. Quantify Diarization Error Rate (DER), Jaccard Error Rate (JER), Missed Speech, False Alarm, and Speaker Confusion rates.

#### Subtasks Breakdown

| Subtask ID | Subtask Name | Actionable Steps & Implementation Scope | Output & Acceptance Criteria |
| :--- | :--- | :--- | :--- |
| `AI-EVAL-02.1` | Benchmark Script Implementation | Implement `scripts/run_diarization_eval.py` utilizing `pyannote.metrics.diarization`. | Executable diarization evaluation script with RTTM parsing capabilities. |
| `AI-EVAL-02.2` | RTTM Comparison & DER Calculation | Run diarization on benchmark audio and compare generated turns against ground-truth RTTM files to calculate Missed Speech, False Alarm, Speaker Confusion, and total DER. | Quantitative DER, Missed Speech, and False Alarm rates against ground-truth intervals. |
| `AI-EVAL-02.3` | Overlapping Speech Assessment | Evaluate the pipeline's precision and recall on frames where multiple speakers speak concurrently. | Overlapping speech detection precision and recall metrics. |
| `AI-EVAL-02.4` | Scorecard Persistence | Persist comprehensive evaluation metrics to `reports/diarization_scorecard.json`. | Structured JSON scorecard containing complete diarization benchmark metrics. |

#### Expected Deliverables & Acceptance Criteria
- **Evaluation Script**: `scripts/run_diarization_eval.py`.
- **Diarization Scorecard**: `reports/diarization_scorecard.json`.
- **Acceptance Gate**:
  - Diarization Error Rate (DER) $\le 18.0\%$.
  - Jaccard Error Rate (JER) $\le 22.0\%$.

#### Checkpoint Specification
- **Checkpoint Required**: **Yes** (Gate M5 Milestone).
- **Checkpoint Type**: Diarization Validation Checkpoint (`diarization-scorecard-v1.0`).
- **Contents**: Metric breakdown JSON, RTTM comparison dumps.
- **Retention**: Permanent.
