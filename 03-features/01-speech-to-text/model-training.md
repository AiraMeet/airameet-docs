# Phase 2 & 3: Model Fine-Tuning Pipeline & Export

## 1. Environment & Dependencies

Model fine-tuning is executed within a containerized Python 3.11 virtual environment equipped with NVIDIA CUDA 12.x drivers.

### Required Dependencies (`requirements-training.txt`)
```text
torch>=2.2.0
torchaudio>=2.2.0
transformers>=4.38.0
datasets>=2.18.0
accelerate>=0.27.0
evaluate>=0.4.1
jiwer>=3.0.3
peft>=0.9.0
bitsandbytes>=0.42.0
ctranslate2>=4.0.0
soundfile>=0.12.1
librosa>=0.10.1
tensorboard>=2.16.0
```

---

## 2. Audio Preprocessing & Feature Extraction

Depending on the model selected during the baseline evaluation:
- **Whisper Pipeline**: Transforms 16 kHz raw audio into an **80-channel Log-Mel Spectrogram** using a 25ms Hanning window with a 10ms hop length, padded or truncated to 30.0 seconds.
- **Wav2Vec 2.0 / Conformer Pipeline**: Normalizes raw 16-bit audio tensors to zero-mean and unit-variance ($\frac{x - \mu}{\sigma}$).

### Data Collator with Dynamic Padding
To optimize training throughput and avoid wasting compute on excessive padding, the pipeline implements a dynamic padding collator that pads audio feature sequences and label token sequences only to the longest sample in each batch.

```python
import torch
from dataclasses import dataclass
from typing import Any, Dict, List, Union

@dataclass
class DataCollatorSpeechSeq2SeqWithPadding:
    processor: Any

    def __call__(self, features: List[Dict[str, Union[List[int], torch.Tensor]]]) -> Dict[str, torch.Tensor]:
        # Extract audio features and pad dynamically
        input_features = [{"input_features": feature["input_features"]} for feature in features]
        batch = self.processor.feature_extractor.pad(input_features, return_tensors="pt")

        # Extract labels and pad dynamically
        label_features = [{"input_ids": feature["labels"]} for feature in features]
        labels_batch = self.processor.tokenizer.pad(label_features, return_tensors="pt")

        # Replace padding token id with -100 to ignore loss calculation
        labels = labels_batch["input_ids"].masked_fill(labels_batch.attention_mask.ne(1), -100)
        
        # If beginning of transcript token is present, strip it
        if (labels[:, 0] == self.processor.tokenizer.bos_token_id).all().cpu().item():
            labels = labels[:, 1:]

        batch["labels"] = labels
        return batch
```

---

## 3. Training Configurations & Hyperparameter Selection

### Recommended Hyperparameter Matrix

| Hyperparameter | Full Fine-Tuning | Parameter-Efficient (LoRA / QLoRA) | Rationale |
| :--- | :--- | :--- | :--- |
| **Base Learning Rate** | $1 \times 10^{-5}$ (Encoder) / $5 \times 10^{-5}$ (Decoder) | $1 \times 10^{-4}$ | Prevents catastrophic forgetting of pretrained weights. |
| **LR Scheduler** | Linear warmup with cosine decay | Linear warmup with cosine decay | Smooth convergence toward global minima. |
| **Warmup Ratio** | 10% of total training steps | 10% of total training steps | Stabilizes early gradient updates. |
| **Per-Device Batch Size** | 4 – 8 samples | 8 – 16 samples | Bound by GPU VRAM capacity. |
| **Gradient Accumulation** | 4 – 8 steps (Effective batch: 32) | 2 – 4 steps (Effective batch: 32) | Simulates large batch stability on single GPUs. |
| **Optimizer** | `AdamW` ($\beta_1=0.9, \beta_2=0.999, \epsilon=10^{-8}$) | `PagedAdamW8bit` | Industry standard for Transformer convergence. |
| **Mixed Precision** | `bf16` (preferred on Ampere+) or `fp16` | `fp16` | $2\times$ memory reduction and $3\times$ tensor core speedup. |
| **Max Gradient Norm** | 1.0 | 1.0 | Clips exploding gradients. |
| **Weight Decay** | 0.01 | 0.01 | Regularization against overfitting on specialized corpus. |

---

## 4. Checkpoint Management & Early Stopping

1. **Evaluation Frequency**: Evaluation is performed every 250 optimization steps on the isolated validation set.
2. **Metric Monitored**: Checkpointing decisions are governed by **Validation Word Error Rate (WER)** rather than cross-entropy loss, ensuring that saved weights directly optimize transcription accuracy.
3. **Checkpoint Retention Policy**:
   - Save only the top 3 best checkpoints based on lowest validation WER.
   - Always retain the latest checkpoint (`checkpoint-latest`) for resumption upon transient hardware interruption.
4. **Early Stopping**: Training automatically terminates if validation WER fails to improve over **5 consecutive evaluation checkpoints** (patience = 5).

---

## 5. Validation Monitoring (WandB & TensorBoard)

The training harness streams real-time telemetry to TensorBoard and Weights & Biases (WandB):
- Training Loss vs. Validation Loss curves.
- Validation WER and CER at each evaluation step.
- Learning rate progression and gradient norm distribution.
- GPU temperature, VRAM consumption, and sample processing throughput (samples/sec).

---

## 6. Model Export & Production Optimization

Once fine-tuning converges, the final checkpoint must be exported from PyTorch training weights into an ultra-low-latency production format.

### Production Runtime: CTranslate2 & INT8 Quantization
CTranslate2 is a dedicated inference engine that provides custom CUDA kernels, memory caching, and 8-bit quantization for Transformer speech models.

#### Export Script (`export_to_ctranslate2.py`)
```bash
# Convert Hugging Face PyTorch checkpoint to CTranslate2 model
ct2-transformers-converter \
    --model /app/checkpoints/best-model \
    --output_dir /app/models/stt-production-int8 \
    --quantization int8_float16 \
    --force
```

### Export Verification Checklist
- [ ] Export directory contains `model.bin`, `vocabulary.json`, `shared_vocabulary.json`, and `config.json`.
- [ ] Inference latency test executes on target GPU with $\text{RTF} \le 0.15$.
- [ ] Quantized model WER on test set exhibits $< 0.3\%$ relative degradation compared to FP32 PyTorch weights.
- [ ] Exported artifact is packaged into a read-only Docker volume for deployment.

---

## 7. Task Specification: `AI-STT-003`

### `AI-STT-003`: Model Fine-Tuning & Quantized Export
- **Task ID**: `AI-STT-003`
- **Task Name**: Acoustic Model Fine-Tuning, Checkpointing & CTranslate2 Export
- **Description**: Implement PyTorch / Hugging Face Trainer fine-tuning pipeline on the curated training corpus. Configure AdamW optimizer, warmup with cosine decay, dynamic padding collator, validation monitoring on WER, and early stopping. Export best checkpoint to CTranslate2 INT8/FP16 format.
- **Priority**: Critical
- **Responsible Department**: AI & Machine Learning
- **Dependencies**: `AI-STT-001`, `AI-STT-002`
- **Implementation Steps**:
  1. Configure `HuggingFace` `Seq2SeqTrainer` or native PyTorch training loop with mixed precision (`bf16`/`fp16`).
  2. Implement data loader with dynamic sequence padding.
  3. Set up evaluation callbacks logging validation loss and WER every 250 steps.
  4. Save top 3 checkpoints and implement early stopping (patience = 5).
  5. Run CTranslate2 export converter on best checkpoint with INT8 quantization.
  6. Measure latency and test set accuracy of exported model artifact.
- **Time Estimates**:
  - Optimistic ($O$): 16h
  - Most Likely ($M$): 24h
  - Pessimistic ($P$): 40h
  - Expected ($E$): $\frac{16 + 4(24) + 40}{6} = 25.33\text{h}$
  - Risk Buffer: 4.0h
- **Acceptance Criteria**: Model achieves $\ge 15\%$ relative WER improvement over baseline on test set; exported CTranslate2 artifact operates with $\text{RTF} \le 0.20$ on target GPU.
- **Deliverable**: `ai-services/training/train_stt.py`, training run logs, and exported `models/stt-production-int8` directory.
- **Status**: `To Do`
