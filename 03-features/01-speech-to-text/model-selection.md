# Phase 2: Model Selection & Pretrained Candidates

## 1. Candidate Architecture Evaluation

Selecting the foundational acoustic model is a critical architectural decision directly affecting transcription accuracy (WER/CER), GPU memory consumption, and real-time streaming latency. 

AiraMeet evaluates three premier deep learning speech families:
1. **OpenAI Whisper** (Encoder-Decoder Transformer)
2. **Meta Wav2Vec 2.0 / MMS** (Self-Supervised Connectionist Temporal Classification - CTC / Cross-Attention)
3. **NVIDIA NeMo ASR Models** (Conformer-CTC / FastConformer-RNNT / Canary)

> [!IMPORTANT]
> **Zero Bias Principle**: OpenAI Whisper is **NOT** pre-selected as the final model. While Whisper exhibits strong zero-shot multilingual generalization, CTC/RNNT Conformer architectures often demonstrate lower streaming latency and lower VRAM footprints. The project mandates a **rigorous baseline zero-shot evaluation** across all candidate models on the held-out test corpus before final model selection.

---

## 2. Deep Dive: Candidate Models Comparison

| Evaluation Dimension | OpenAI Whisper (`small` / `medium`) | Meta Wav2Vec 2.0 (`XLSR-53` / `large`) | NVIDIA NeMo (`FastConformer-CTC / RNNT`) |
| :--- | :--- | :--- | :--- |
| **Model Architecture** | Sequence-to-Sequence (Encoder-Decoder Transformer) | Transformer Encoder with linear CTC projection head | Conformer (Convolution-augmented Transformer) CTC/RNN-T |
| **Acoustic Input Representation** | 80-channel Log-Mel Spectrogram (25ms window, 10ms hop) | Raw uncompressed 16 kHz audio waveform | 80-channel Log-Mel Spectrogram (4x-8x temporal subsampling) |
| **Parameter Count** | `small`: 244M \| `medium`: 769M \| `large-v3`: 1,550M | `XLSR-53`: 317M \| `large`: 315M | `FastConformer-CTC`: 115M \| `Canary`: 1,000M |
| **Inference VRAM (FP16 / INT8)** | `small`: ~1.8 GB / ~1.0 GB \| `medium`: ~4.5 GB / ~2.2 GB | ~1.5 GB / ~0.8 GB | ~1.2 GB / ~0.7 GB (Very low footprint) |
| **Training VRAM (Full Fine-Tune)** | 16 GB – 24 GB (or 8–12 GB with LoRA/QLoRA) | 12 GB – 16 GB | 12 GB – 16 GB |
| **Streaming Feasibility** | Moderate (chunked sliding window with prefix prompting) | High (natively supports chunked causal streaming CTC) | Very High (FastConformer engineered specifically for streaming) |
| **Inference Latency / RTF** | RTF: ~0.15 – 0.25 (Good with CTranslate2) | RTF: ~0.08 – 0.15 (Fast, non-autoregressive) | RTF: ~0.05 – 0.12 (Ultra-fast non-autoregressive) |
| **Multilingual & Dialect Support** | State-of-the-art across 99+ languages, including Arabic | Excellent cross-lingual pretraining (53+ languages) | High for English/Spanish/German/Arabic (Canary / Conformer) |
| **Inherent Punctuation & Casing** | Native (predicts punctuation & capitalization natively) | Absent in basic CTC (requires external language model) | Conformer-CTC requires separate punctuation/capitalization model |
| **License** | MIT License (Permissive commercial use) | MIT / Apache 2.0 (Permissive commercial use) | Apache 2.0 (Permissive commercial use) |

---

## 3. Comparative Trade-Off Analysis

```mermaid
quadrantChart
    title Speech Model Selection Trade-Offs
    x-axis Low Streaming Latency --> High Streaming Latency
    y-axis Low Acoustic Accuracy (WER) --> High Acoustic Accuracy (WER)
    quadrant-1 High Accuracy, High Latency
    quadrant-2 High Accuracy, Low Latency (Target)
    quadrant-3 Low Accuracy, Low Latency
    quadrant-4 Low Accuracy, High Latency
    "OpenAI Whisper small": [0.65, 0.88]
    "OpenAI Whisper medium": [0.80, 0.94]
    "NVIDIA FastConformer-CTC": [0.25, 0.85]
    "Meta Wav2Vec2-XLSR": [0.35, 0.78]
```

### Strengths & Weaknesses Breakdown

#### 1. OpenAI Whisper (`whisper-small` / `whisper-medium`)
- **Pros**:
  - Predicts punctuation, capitalization, and inverse text normalization (numbers, dates) natively.
  - Highly robust to background noise and varied microphone acoustics.
  - Supported natively by `CTranslate2` and `TensorRT-LLM` for $4\times$ inference acceleration.
- **Cons**:
  - Autoregressive decoder can occasionally hallucinate or loop during prolonged background noise.
  - Higher latency per token generation compared to non-autoregressive CTC models.

#### 2. Meta Wav2Vec 2.0 (`wav2vec2-large-xlsr-53`)
- **Pros**:
  - CTC architecture evaluates all time-steps simultaneously without autoregressive looping.
  - Zero hallucination risk during silence.
  - Very fast inference with low computational complexity.
- **Cons**:
  - Outputs raw lowercase text without punctuation or capitalization; requires an external secondary NLP restoration model.
  - Sensitivity to out-of-vocabulary technical terms without a large language model decoder (e.g., KenLM).

#### 3. NVIDIA NeMo (`FastConformer-CTC`)
- **Pros**:
  - State-of-the-art combination of self-attention and depthwise separable convolutions.
  - Extreme computational efficiency via $8\times$ depthwise subsampling.
  - Native streaming cache mechanisms for sub-second latency.
- **Cons**:
  - Deployment dependency on NVIDIA NeMo / PyTriton ecosystem.
  - Requires post-processing for Arabic diacritics and capitalization.

---

## 4. Model Selection Methodology & Gate Decision

To ensure a completely objective, data-driven decision:
1. **Step 1 (Week 2, Day 1–2)**: AI engineers set up benchmarking scripts for all three candidate families.
2. **Step 2 (Week 2, Day 3–4)**: Execute zero-shot evaluation on the held-out test split of the AiraMeet corpus.
3. **Step 3 (Week 2, Day 5)**: Compile the **Benchmark Scorecard**:
   - Word Error Rate (WER) and Character Error Rate (CER).
   - Real-Time Factor (RTF) on target GPU hardware.
   - Peak VRAM footprint under concurrent loads.
4. **Step 4 (Milestone M2 Gate)**: Architecture Board and AI Lead review scorecard and formally sign off on the single model selected for custom fine-tuning in Phase 3.

---

## 5. Task Specification: `AI-STT-002`

### `AI-STT-002`: Zero-Shot Model Benchmarking & Selection Gate
- **Task ID**: `AI-STT-002`
- **Task Name**: Pretrained Model Benchmarking & Baseline Evaluation
- **Description**: Configure automated zero-shot evaluation harness for candidate models (OpenAI Whisper, Meta Wav2Vec 2.0, NVIDIA NeMo Conformer). Measure WER, CER, RTF, and VRAM memory footprint on the held-out test split. Compile comparative scorecard to guide model selection.
- **Priority**: Critical
- **Responsible Department**: AI & Machine Learning
- **Dependencies**: `AI-STT-001`
- **Implementation Steps**:
  1. Set up evaluation harness scripts utilizing Hugging Face `evaluate` and `jiwer`.
  2. Load pretrained weights for Whisper (`small`, `medium`), Wav2Vec 2.0 (`XLSR-53`), and NeMo Conformer.
  3. Run batch evaluation on test split, recording execution time and memory peak.
  4. Generate comparative evaluation scorecard markdown matrix.
  5. Conduct Milestone M2 review meeting and finalize model choice for fine-tuning.
- **Time Estimates**:
  - Optimistic ($O$): 10h
  - Most Likely ($M$): 16h
  - Pessimistic ($P$): 26h
  - Expected ($E$): $\frac{10 + 4(16) + 26}{6} = 16.67\text{h}$
  - Risk Buffer: 3.0h
- **Acceptance Criteria**: Baseline evaluation completed across all candidate models on identical test data; scorecard published with measured baseline WER, CER, and RTF; formal model selection sign-off achieved.
- **Deliverable**: `ai-services/scripts/benchmark_candidates.py` and evaluation scorecard report.
- **Status**: `To Do`
