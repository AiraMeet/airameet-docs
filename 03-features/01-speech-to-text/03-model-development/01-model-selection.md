# 01. Model Development — Model Selection & Architecture Decision Record (ADR)

## 1. Overview & Context

AiraMeet requires an Automatic Speech Recognition (ASR) foundation model and a Speaker Diarization architecture capable of transcribing conversational Egyptian Arabic, English, and technical code-switching, identifying speaker turns, and executing with low latency on target GPU hardware.

---

## 2. ASR Candidate Comparison Matrix

| Evaluation Dimension | OpenAI Whisper (`medium`) | Meta MMS-1B / XLSR-53 | NVIDIA NeMo FastConformer |
| :--- | :--- | :--- | :--- |
| **Model Architecture** | Sequence-to-Sequence Encoder-Decoder | Non-Autoregressive CTC | Hybrid Conformer CTC / RNN-T |
| **Arabic & Egyptian Dialect**| **High**: Broad exposure to colloquial Arabic in its 680,000-hour corpus. | **Moderate**: Strong phonemes; weak dialect syntax without external LM. | **Good**: High on MSA; sparse Egyptian colloquial data out of the box. |
| **English & Code-Switching** | **Superior**: Unified bilingual BPE tokenizer natively decodes mixed scripts. | **Poor**: Lacks unified bilingual vocabulary; needs dual language models. | **Moderate**: Requires custom bilingual vocabulary retraining. |
| **Punctuation & Casing** | **Built-in**: Emits capitalized English, Arabic punctuation, and clean text. | **None**: Emits unpunctuated lowercase text; requires heavy secondary models. | **Requires LM**: Requires dedicated punctuation restoration model. |
| **Inference Latency & RTF** | **Medium**: RTF ~0.12 with CTranslate2 INT8 on RTX 4090. | **Ultra-Fast**: RTF ~0.04 via non-autoregressive decoding. | **Fast**: RTF ~0.07 via 8x downsampling and TensorRT. |
| **GPU Memory Footprint** | ~2.2 GB VRAM in INT8 quantization. | ~1.8 GB VRAM. | ~3.5 GB VRAM. |
| **Fine-Tuning Ecosystem** | **High**: Native Hugging Face `Seq2SeqTrainer`, LoRA/PEFT, PyTorch. | **Moderate**: Hugging Face CTC trainer, but complex token alignment. | **Complex**: NeMo framework lock-in, PyTorch Lightning dependencies. |
| **Serving & Deployment** | **Standard**: Seamless CTranslate2 / ONNX export for high-throughput serving. | Custom ONNX / TensorRT execution. | NVIDIA Triton Inference Server with NeMo backend. |

---

## 3. Architecture Decisions & Diarization Strategy

1. **ASR Model**: Selected **`openai/whisper-medium`** (769M parameters).
   - Unified BPE vocabulary prevents phonetic drift when English technical terms appear in Arabic speech.
   - Built-in punctuation eliminates secondary post-processing latency.
   - Standard CTranslate2 INT8 conversion reduces VRAM to 2.2 GB with an RTF of 0.12.
2. **Diarization Strategy**: Selected **PyAnnote.audio 3.1 (Cascaded Pipeline)**.
   - Integrated end-to-end models (e.g., EEND) cannot be trained on the current dataset (`speaker` is `unknown`) and lack pretrained bilingual Egyptian checkpoints.
   - PyAnnote 3.1 provides state-of-the-art zero-shot acoustic segmentation, embedding clustering, and overlapping speech detection.
3. **Execution Boundary**: Diarization operates on continuous meeting audio to maintain cross-turn voiceprint clusters, while Whisper processes 1.5s sliding window chunks.

---

## 4. Engineering Tasks & Subtasks Breakdown

---

### Task `AI-ASR-01`: Candidate Baseline Evaluation & Architecture Benchmark

| Attribute | Specification |
| :--- | :--- |
| **Task ID** | `AI-ASR-01` |
| **Task Name** | Candidate Baseline Evaluation & Architecture Benchmark |
| **Phase** | AI Phase 1 / Phase 3 (Model Selection & Baseline) |
| **Priority / Owner** | `High` / AI Engineering Team |
| **Dependencies** | `AI-DS-03` (Leak-Free Test Split Manifest) |
| **Checkpoint** | **Yes** — Gate M2 Baseline Artifact (`reports/baseline_scorecard.json`) |

#### Objective & Scope
Perform a rigorous zero-shot baseline evaluation of `openai/whisper-medium` on the isolated held-out test split (`test.jsonl`). Establish ground-truth baseline Word Error Rate (WER) and Character Error Rate (CER) broken down across pure Arabic, pure English, and code-switched audio. Measure baseline inference speed (RTF) and peak VRAM consumption on target GPU hardware prior to fine-tuning.

#### Subtasks Breakdown

| Subtask ID | Subtask Name | Actionable Steps & Implementation Scope | Output & Acceptance Criteria |
| :--- | :--- | :--- | :--- |
| `AI-ASR-01.1` | Environment & Dependencies Setup | Install PyTorch 2.2+, CUDA 12.x, `transformers`, `torchaudio`, and `jiwer`. Initialize raw `openai/whisper-medium` weights on dedicated GPU hardware. | Verified GPU PyTorch runtime; downloaded foundation Whisper-Medium checkpoint. |
| `AI-ASR-01.2` | Evaluation Harness Implementation | Implement `scripts/run_baseline_eval.py` to stream through `data/manifests/test.jsonl` using standardized greedy decoding parameters (`temperature=0.0`, `language="ar"`). | Working evaluation harness script with progress tracking and batch execution. |
| `AI-ASR-01.3` | Stratified Scoring & Normalization | Apply standardized bilingual text normalization (`src/dataset/text_normalizer.py`) to both hypotheses and references. Compute overall WER, CER, pure Egyptian Arabic WER, pure English WER, and code-switching WER. | Stratified error metrics computed using `jiwer` without script or casing bias. |
| `AI-ASR-01.4` | Hardware Profiling & Scorecard Persistence | Measure peak GPU VRAM via `torch.cuda.max_memory_allocated()`, calculate average Real-Time Factor (RTF), and persist output to `reports/baseline_scorecard.json`. | Structured JSON scorecard containing all baseline accuracy and compute metrics. |

#### Expected Deliverables & Acceptance Criteria
- **Evaluation Script**: `scripts/run_baseline_eval.py`.
- **Scorecard Report**: `reports/baseline_scorecard.json` containing detailed error rates per language stratum and compute metrics.
- **Acceptance Gate**: Baseline performance quantitatively documented on identical test partition (expected baseline WER: ~25%–30% on colloquial Egyptian speech) as a reference point for fine-tuning validation.

#### Checkpoint Specification
- **Checkpoint Required**: **Yes** (Gate M2 Milestone).
- **Checkpoint Type**: Permanent Evaluation Record (`asr-baseline-eval`).
- **Contents**: `baseline_scorecard.json` + raw hypothesis text dump + commit SHA hash.
- **Retention**: Permanent.

---

### Task `AI-ASR-01B`: Diarization Zero-Shot Baseline Validation on Multi-Speaker Data

| Attribute | Specification |
| :--- | :--- |
| **Task ID** | `AI-ASR-01B` |
| **Task Name** | Diarization Zero-Shot Baseline Validation on Multi-Speaker Data |
| **Phase** | AI Phase 1 / Phase 5 (Speaker Diarization Research) |
| **Priority / Owner** | `Medium` / AI Engineering Team |
| **Dependencies** | None |
| **Checkpoint** | **No** (Exploratory validation step) |

#### Objective & Scope
Verify the out-of-the-box zero-shot performance of `pyannote/speaker-diarization-3.1` on multi-speaker meeting recordings. Measure initial Diarization Error Rate (DER) to validate that pretrained voiceprint embeddings can distinguish speaker turns before coupling with ASR.

#### Subtasks Breakdown

| Subtask ID | Subtask Name | Actionable Steps & Implementation Scope | Output & Acceptance Criteria |
| :--- | :--- | :--- | :--- |
| `AI-ASR-01B.1` | PyAnnote Initialization & Token Authentication | Install `pyannote.audio` and configure Hugging Face token authentication to download pretrained segmentation and embedding models. | Authenticated environment with PyAnnote 3.1 weights initialized on CUDA device. |
| `AI-ASR-01B.2` | Multi-Speaker Sample Assembly | Assemble 5 sample meeting audio recordings (3–5 concurrent speakers) with corresponding ground-truth RTTM annotation files. | Curated RTTM ground-truth dataset and corresponding 16 kHz audio files. |
| `AI-ASR-01B.3` | Zero-Shot Benchmark & DER Scoring | Implement `scripts/run_diarization_eval.py` to run segmentation, embedding extraction, and clustering, scoring DER, Missed Speech, and False Alarm rates via `pyannote.metrics`. | Baseline diarization scorecard `reports/diarization_baseline.json` with DER score. |

#### Expected Deliverables & Acceptance Criteria
- **Evaluation Script**: `scripts/run_diarization_eval.py`.
- **Benchmark Output**: `reports/diarization_baseline.json`.
- **Acceptance Gate**: Diarization Error Rate (DER) $\le 18.0\%$ achieved on multi-speaker validation audio.

#### Checkpoint Specification
- **Checkpoint Required**: **No**. Internal verification result documented directly in evaluation report.
