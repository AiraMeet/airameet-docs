# 03. Model Development — Candidate Selection

## Purpose

Choose a multilingual ASR candidate for Egyptian Arabic, English and Arabic-English code-switching using reproducible quality, streaming and operational evidence. No model is selected in the available project record.

## Scope and prerequisites

Compare pretrained candidates and decide which, if any, to fine-tune. Requires a reviewed dataset split, evaluation normalization policy, target hardware/workload and license review. See [dataset pipeline](../02-data-engineering/dataset-pipeline.md) and [evaluation](evaluation.md).

## Candidates and considerations

The source material names Whisper, Meta Wav2Vec 2.0/XLSR or MMS, and NVIDIA NeMo Conformer/Canary families. These are a shortlist to investigate, not endorsed versions or measured rankings.

| Candidate family | Potential strengths to verify | Risks/questions to verify |
|---|---|---|
| Whisper encoder-decoder | Multilingual transcription, punctuation; chunked inference may be usable. | Chunk boundary stability, Egyptian dialect/code-switch accuracy, hallucination behavior on silence, runtime/license for exact checkpoint and distribution. |
| Wav2Vec 2.0 / MMS CTC | CTC decoding may suit low-latency streaming; multilingual pretrained options exist. | Exact Arabic/dialect coverage, decoding/language model and punctuation needs, timestamp semantics, license and deployment. |
| NeMo Conformer / related ASR | Streaming-oriented configurations may be available; deployment ecosystem. | Exact model language coverage and checkpoint license; integration complexity and supported runtime. |

Do not copy historical parameter counts, VRAM, latency, accuracy or license claims into a decision record without checking the exact artifact/version and measuring it. Confirm commercial use and redistribution from the checkpoint's own license terms.

## Technical approach

Use the same untouched test samples and reference normalization for baseline comparisons. Use validation only for candidate iteration. Report separate results for Egyptian Arabic, English and code-switched speech, plus aggregate results; segment by noise/device/accent only where labels and sample sizes support meaningful conclusions. Include streaming behavior, not only offline batch accuracy.

## Implementation steps

1. Freeze candidate identifiers, artifact revisions, tokenizer/processor and license evidence.
2. Implement a common input adapter and decode configuration; record language prompts/decoding settings per candidate.
3. Measure WER/CER with language-aware normalization, real-time factor, time-to-first-partial, finalization latency, memory and concurrency on target hardware.
4. Inspect errors, particularly script switching, names, numerals, overlap, silence hallucinations and long utterances.
5. Publish results with sample counts, confidence/limitations and reproducible command/config.
6. Obtain AI lead/product/architecture approval and record the chosen model/runtime in a decision record before training.

## Tools and technologies

Python evaluation harness; candidate-native libraries (for example Transformers or NeMo); `jiwer` or equivalent metrics after normalization review; hardware telemetry. Exact package/runtime versions are TBD and must be locked for each run.

## Deliverables

Candidate manifest and license review; benchmark scripts/config; measured scorecard; error analysis; signed model-selection decision.

## Acceptance criteria

All candidates use the same data and comparable settings; no test-set tuning; language-specific results and compute measurements are reported; selection rationale addresses quality, streaming, operations and licensing; no unmeasured value is presented as benchmark evidence.

## Dependencies and open questions

Depends on data release, target GPU/CPU environment and product latency/accuracy gates. Open: precise checkpoint candidates/revisions, runtime constraints, decoding policy, deployment license approval, and whether one model can adequately cover all three speech conditions.
