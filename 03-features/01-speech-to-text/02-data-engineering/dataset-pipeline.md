# 02. Data Engineering — Dataset Pipeline

## 1. Overview & Data Architecture

The dataset pipeline ingests raw conversational audio and metadata, validates acoustic formats, executes dual-script normalization, prevents data leakage between splits, and produces immutable manifests for fine-tuning Whisper on Egyptian Arabic and English code-switching.

---

## 2. Dataset Structure & Directives

```csv
meeting,speaker,audio_path,text,start,end,duration
train,unknown,audio/train/000001.wav,طيب، بتسمع اغاني طبعا,0,2.91,2.91
train,unknown,audio/train/000002.wav,"وهي إنها بتطلّع Layer أو طبقة جديدة لجذع الشجرة من برة، كل سنة أو كل موسم.",0,5.307,5.307
train,unknown,audio/train/000003.wav,زي، يا عزيزي، إيليفين في Stranger Things.,0,1.615,1.615
train,unknown,audio/train/000004.wav,"يسمّونها بالإنقليزي (You have to recuse yourself)",0,2.425,2.425
```

### Architectural Directives
1. **Handling the `unknown` Speaker Column**:
   - The current `speaker` column contains `unknown`.
   - **Supervised Diarization Constraint**: Cannot train supervised diarization models with this dataset. Pretrained zero-shot PyAnnote 3.1 is used instead.
   - **ASR Usability**: The dataset is completely valid for supervised ASR acoustic and language model fine-tuning.
2. **Preventing Acoustic Data Leakage**:
   - Splitting must be grouped by recording session / source ID so that all segments from the same meeting stay in exactly one split.
3. **Relative Paths**:
   - Audio paths are relative (e.g., `audio/train/000001.wav`) and resolved via `DATASET_ROOT_DIR`.

---

## 3. Engineering Tasks & Subtasks Breakdown

---

### Task `AI-DS-01`: Dataset Inspection, Audio Format Harmonization & Quality Audit

| Attribute | Specification |
| :--- | :--- |
| **Task ID** | `AI-DS-01` |
| **Task Name** | Dataset Inspection, Audio Format Harmonization & Quality Audit |
| **Phase** | AI Phase 2: Dataset Engineering |
| **Priority / Owner** | `Critical` / AI Engineering Team |
| **Dependencies** | None |
| **Checkpoint** | **Yes** — Gate M1 Data Audit Checkpoint (`reports/inspection_report.json`) |

#### Objective & Scope
Audit all audio recordings and metadata. Standardize audio sample rates to 16,000 Hz Mono WAV (16-bit PCM). Detect and quarantine corrupted, unreadable, or duplicate files. Enforce duration limits ($0.5\text{s} \le \text{duration} \le 30.0\text{s}$) to prevent Whisper memory allocation faults, and isolate noisy samples via WADA-SNR analysis.

#### Subtasks Breakdown

| Subtask ID | Subtask Name | Actionable Steps & Implementation Scope | Output & Acceptance Criteria |
| :--- | :--- | :--- | :--- |
| `AI-DS-01.1` | Schema & Path Integrity Validation | Implement CSV parser in `src/dataset/audio_validator.py` verifying columns (`meeting,speaker,audio_path,text,start,end,duration`) and checking that relative audio paths exist on disk. | Validated CSV records; throws explicit error on missing audio paths or malformed columns. |
| `AI-DS-01.2` | Audio Decoding & Format Conversion | Read audio using `soundfile` and `torchaudio`. Convert stereo or non-standard audio to uncompressed 16,000 Hz Mono 16-bit linear PCM and save to `data/processed/`. | Standardized 16 kHz Mono 16-bit PCM WAV files saved to `data/processed/`. |
| `AI-DS-01.3` | Duration & Acoustic Quality Filtering | Check for NaN/Inf sample values, digital clipping ($> 0.99$ saturation), compute WADA-SNR, and discard utterances $< 0.5\text{s}$ or $> 30.0\text{s}$. Apply peak normalization to -1.0 dBFS. | Corrupted/clipped audio isolated; duration strictly bounded between 0.5s and 30.0s. |
| `AI-DS-01.4` | Quarantine Ledger & Audit Report Generation | Author `scripts/run_inspection.py` to process the entire corpus and generate `reports/inspection_report.json` detailing valid, quarantined, and discarded files with explicit reason codes. | Comprehensive audit report `reports/inspection_report.json` and quarantine ledger. |

#### Expected Deliverables & Acceptance Criteria
- **Inspection Tooling**: `src/dataset/audio_validator.py`, `scripts/run_inspection.py`.
- **Harmonized Audio**: `data/processed/*.wav` (16 kHz Mono PCM).
- **Audit Ledger**: `reports/inspection_report.json` detailing valid, quarantined, and discarded files.
- **Acceptance Gate**: 100% of accepted files successfully decode without exceptions; all corrupted files quarantined with explicit reason codes.

#### Checkpoint Specification
- **Checkpoint Required**: **Yes** (Gate M1 Milestone).
- **Checkpoint Type**: Data Audit Checkpoint (`data-audit-v1.0`).
- **Contents**: `inspection_report.json` + quarantine ledger.
- **Retention**: Permanent.

---

### Task `AI-DS-02`: Arabic & English Code-Switching Text Normalization

| Attribute | Specification |
| :--- | :--- |
| **Task ID** | `AI-DS-02` |
| **Task Name** | Arabic & English Code-Switching Text Normalization |
| **Phase** | AI Phase 2: Dataset Engineering |
| **Priority / Owner** | `Critical` / AI Engineering Team |
| **Dependencies** | `AI-DS-01` (Harmonized Audio) |
| **Checkpoint** | **No** (Intermediate data transformation) |

#### Objective & Scope
Construct a dual-script text normalization pipeline for colloquial Egyptian Arabic mixed with English technical jargon. Clean and standardize Arabic orthography while strictly preserving English words, character casing, and alphanumeric technical terms without phonetic transliteration.

#### Subtasks Breakdown

| Subtask ID | Subtask Name | Actionable Steps & Implementation Scope | Output & Acceptance Criteria |
| :--- | :--- | :--- | :--- |
| `AI-DS-02.1` | Unicode Canonicalization & Whitespace Cleanup | Implement Unicode NFC normalization (`unicodedata.normalize('NFC', text)`) and collapse redundant spaces and control characters in `src/dataset/text_normalizer.py`. | Standardized UTF-8 strings without hidden formatting or control bytes. |
| `AI-DS-02.2` | Arabic Diacritic & Orthographic Standardization | Strip all Arabic Tashkeel (`[\u064B-\u0652]`, Harakat & Tanween), fold Alef variations (`[أ, إ, آ, ٱ]` $\rightarrow$ `ا`), map Alif Maqsura (`ى` $\rightarrow$ `ي`), standardize Taa Marbuta (`ة` $\rightarrow$ `ه`), and delete Tatweel (`\u0640`). | Canonical Arabic text with uniform letter forms and stripped vocalization diacritics. |
| `AI-DS-02.3` | Code-Switching Protection & Punctuation Filtering | Preserve all Latin alphanumeric sequences (`[A-Za-z0-9]+`) with exact casing (e.g., `"Layer"`, `"Stranger Things"`). Strip non-syntactic brackets and quotes. | English technical vocabulary preserved intact without transliteration or case distortion. |
| `AI-DS-02.4` | Unit Regression Suite & Cleaned Manifest Export | Author `tests/unit/test_text_normalizer.py` verifying 50 bilingual fixtures; export normalized dataset to `data/cleaned_metadata.csv`. | 100% passing tests for bilingual fixtures; exported clean metadata CSV. |

#### Expected Deliverables & Acceptance Criteria
- **Normalizer Module**: `src/dataset/text_normalizer.py`.
- **Cleaned Dataset**: `data/cleaned_metadata.csv`.
- **Regression Tests**: `tests/unit/test_text_normalizer.py`.
- **Acceptance Gate**: 100% pass rate across bilingual test fixtures; zero dropped English technical terms.

#### Checkpoint Specification
- **Checkpoint Required**: **No**. Intermediate pipeline output verified via unit test assertions.

---

### Task `AI-DS-03`: Leak-Free Stratified Group Splitting & Manifest Generation

| Attribute | Specification |
| :--- | :--- |
| **Task ID** | `AI-DS-03` |
| **Task Name** | Leak-Free Stratified Group Splitting & Manifest Generation |
| **Phase** | AI Phase 2: Dataset Engineering |
| **Priority / Owner** | `Critical` / AI Engineering Team |
| **Dependencies** | `AI-DS-02` (Normalized Transcripts) |
| **Checkpoint** | **Yes** — Gate M1 Data Release Milestone (`data/manifests/CHECKSUMS.sha256`) |

#### Objective & Scope
Partition the cleaned dataset into Training (80%), Validation (10%), and Test (10%) splits. Strictly enforce grouped splitting by recording session / meeting ID to prevent acoustic leakage between partitions. Generate immutable JSONL manifests and SHA256 cryptographic checksums.

#### Subtasks Breakdown

| Subtask ID | Subtask Name | Actionable Steps & Implementation Scope | Output & Acceptance Criteria |
| :--- | :--- | :--- | :--- |
| `AI-DS-03.1` | Recording Session Grouping | Implement `src/dataset/manifest_builder.py` reading `data/cleaned_metadata.csv` and aggregating utterances by `meeting` identifier. | Grouped dictionary mapping meeting IDs to all child audio segments. |
| `AI-DS-03.2` | Stratified 80/10/10 Split Partitioning | Execute group-based splitting via `sklearn.model_selection.GroupShuffleSplit` with fixed pseudo-random seed `42`. | Deterministic 80% train, 10% validation, 10% test split partitions. |
| `AI-DS-03.3` | Leakage Assertion Verification | Assert programmatically that the set intersection of recording session IDs between train and test is strictly empty ($\emptyset$). | Automated assertion confirming zero acoustic leakage across splits. |
| `AI-DS-03.4` | Manifest & Checksum Generation | Write JSONL files (`train.jsonl`, `val.jsonl`, `test.jsonl`), calculate SHA256 hashes into `data/manifests/CHECKSUMS.sha256`, and author `scripts/generate_stats.py` to emit `reports/dataset_stats.json`. | Immutable `train.jsonl`, `val.jsonl`, `test.jsonl` manifests, checksums, and dataset stats report. |

#### Expected Deliverables & Acceptance Criteria
- **Manifest Files**: `data/manifests/train.jsonl`, `data/manifests/val.jsonl`, `data/manifests/test.jsonl`.
- **Checksums**: `data/manifests/CHECKSUMS.sha256`.
- **Statistics Report**: `reports/dataset_stats.json` (hours per split, duration distributions, vocabulary counts).
- **Acceptance Gate**: Zero overlapping recording sessions between splits; immutable SHA256 hashes generated.

#### Checkpoint Specification
- **Checkpoint Required**: **Yes** (Gate M1 Milestone).
- **Checkpoint Type**: Data Partitioning Release Checkpoint (`data-v1.0`).
- **Contents**: Manifest files, SHA256 digests, and dataset statistics report.
- **Retention**: Permanent.
