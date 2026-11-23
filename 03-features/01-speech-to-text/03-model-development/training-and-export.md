# 03. Model Development — Training and Export

## Purpose

Describe a reproducible fine-tuning lifecycle for the approved candidate and a controlled path from training checkpoint to deployable inference artifact. Values below are starting proposals, not an approved recipe or achieved result.

## Scope and prerequisites

Requires approved [model selection](model-selection.md), versioned [dataset release](../02-data-engineering/dataset-pipeline.md), target hardware, license clearance and evaluation gates. Model training does not own online meeting routing or participant identity.

## Technical approach

- Pin framework, CUDA/runtime, model revision, tokenizer/processor, data release, seed and full config.
- Select preprocessing from the chosen model's documented contract. Prior alternatives included Whisper log-mel features and waveform normalization for CTC-style models; do not mix assumptions across architectures.
- Keep speaker/session-grouped validation separate from training and test. Choose checkpoints using validation metrics only.
- Record every run (code revision, configuration, data manifest hash, hardware, logs, checkpoints and evaluation outputs). Do not log personal data or raw transcript samples in public experiment trackers.
- Compare full fine-tuning and parameter-efficient methods only if supported by candidate/runtime and dataset size. Hyperparameters are tuned on validation, not test.
- Export/quantize only after model selection. Re-run quality and latency evaluation on the exported artifact; quantization drift must be measured.

## Implementation steps

1. Build a versioned loader and model-specific preprocessing path with dynamic batching/padding where supported.
2. Establish a reproducible baseline training run. Track training/validation loss, WER/CER by language, throughput, GPU memory and early-stop reason.
3. Tune a bounded set of optimizer, learning-rate, precision, batch/accumulation and regularization settings using validation data.
4. Save resumable checkpoints and a documented best-checkpoint rule; retain artifacts per project storage policy.
5. Export to the selected serving runtime only if conversion is supported for the exact checkpoint. Record quantization mode and converter versions.
6. Run artifact load, deterministic smoke, regression, language-slice and performance checks; publish model card and artifact checksum.

## Initial experiment ranges (proposal only)

An earlier draft suggested AdamW, linear warmup/cosine decay, warmup near 10%, gradient clipping at 1.0, weight decay near 0.01, mixed precision (`bf16` where supported or `fp16`), and effective batch size around 32 using accumulation. It suggested learning rates from `1e-5` to `5e-5` for full tuning and around `1e-4` for adapter tuning, with batch sizes adjusted to accelerator memory. Treat these solely as a search-space starting point: suitable settings depend on model architecture, corpus scale and optimizer implementation. Earlier fixed evaluation cadence/early-stopping patience values (250 steps and 5 checks) likewise require training-set-size review.

## Tools and technologies

Prior draft proposed Python 3.11, PyTorch, torchaudio, Transformers, Datasets, Accelerate, Evaluate, jiwer, PEFT/bitsandbytes and CTranslate2. Treat this list as candidate tooling, not a committed dependency set. NeMo and other model families may require different training/export paths. Pin compatible versions after selection.

## Deliverables

Training code/config; run manifest and logs; selected checkpoint; export/conversion procedure; final model artifact and checksum; model card including languages, limitations, license, training data reference, evaluation and rollback version.

## Acceptance criteria

Training is reproducible from recorded inputs; checkpoint selection uses validation only; held-out test is evaluated once per release decision; exported artifact loads in the serving environment; artifact quality/runtime are measured against gates in [evaluation](evaluation.md); data/artifact access follows policy.

## Dependencies and open questions

Depends on selected architecture, data release, compute budget and serving runtime. TBD: training strategy, hyperparameters, checkpoint retention, experiment tracking service, export format, quantization policy and artifact registry.
