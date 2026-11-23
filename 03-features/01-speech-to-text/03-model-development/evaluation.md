# 03. Model Development — Evaluation Protocol

## Purpose

Define comparable quality and runtime measurements for model selection, training and release. No AiraMeet benchmark measurements are currently available in this documentation.

## Scope and prerequisites

Applies to offline candidate evaluation and exported model verification. Requires an approved, held-out test release, reference text policy, target hardware/workload and candidate artifact/config. See [requirements](../01-planning-design/requirements.md).

## Metrics

- **WER**: `(substitutions + deletions + insertions) / reference word count`.
- **CER**: character edit distance divided by reference character count.
- **Relative WER change**: `(baseline WER - candidate WER) / baseline WER`; report absolute scores too.
- **RTF**: inference processing time divided by input audio duration; lower is faster than audio playback time.
- **Latency**: report time-to-first-partial and time-to-final separately, with p50/p95 and defined measurement boundaries.
- **Capacity/resource use**: concurrent streams, queue delay, dropped/late frames and peak accelerator/host memory under a specified workload.

For Arabic and code-switching, freeze text normalization before scoring. Report raw and normalized scoring where useful; state tokenization, punctuation, diacritic, numeral and script rules. Report per-language and code-switch slices as well as aggregate results, with denominators. Do not compare figures calculated with different references or normalization.

## Technical approach and protocol

1. Use validation for iteration and reserve test for a release/selection gate. Ensure group separation in [dataset pipeline](../02-data-engineering/dataset-pipeline.md).
2. Freeze test manifest hash, model revision, decoding settings, runtime/hardware and metric package versions.
3. Evaluate clean, noisy, device and code-switch conditions only where the corpus has defensible labels or controlled test construction.
4. Profile end-to-end streaming with representative concurrency, including VAD, queueing, network/service overhead and client-visible finalization; separately report model-only inference.
5. Analyze error classes (dialect vocabulary, code-switch boundaries, names, numerals, overlap, silence hallucinations, segmentation/timestamps).
6. Compare pretrained and fine-tuned/exported artifacts on identical test samples. Preserve raw run outputs for audit.

## Current targets and decision gates

| Measure | Existing proposed target | Status |
|---|---|---|
| End-to-end visible latency | p95 ≤ 3.0 s | Target in requirements; workload and boundary must be agreed. |
| Model RTF | ≤ 0.25 | Proposed target; measure on specified hardware. |
| Fine-tuning effect | ≥15% relative WER improvement over baseline | Proposed gate; validate after baseline and dataset review. |
| Absolute WER/CER | TBD | Set after baseline, corpus audit and product review. |
| Noisy speech quality | Approx. 10 dB SNR was cited | Test condition proposal; acceptable quality threshold TBD. |
| Concurrent streams | 16 per worker was cited | Capacity hypothesis; load test required. |

Earlier drafts also listed p95 chunk latency, VRAM and quantization degradation limits, but values differed between documents and lacked run evidence. Re-establish these from target hardware instead of treating them as approved gates.

## Tools and technologies

Python harness, `jiwer`/equivalent WER-CER implementation, model runtime timers and GPU/host telemetry are candidate tools. Pin versions and provide a reproducible command/config in each result.

## Deliverables

Evaluation plan; frozen test and normalization references; result tables by language/condition; performance report; error analysis; signed gate decision with artifact IDs.

## Acceptance criteria

Every reported result can be reproduced and traces to exact data/model/config; no test data is used for tuning; metrics include sample counts and slices; thresholds are approved before a release decision; blank/TBD values are not presented as passes.

## Dependencies and open questions

Depends on data governance, corpus labels, product quality thresholds and target deployment hardware. Define dialect labels, scoring normalization, confidence intervals, load profile, latency budget breakdown and acceptance process.
