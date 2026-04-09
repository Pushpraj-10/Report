# Dynamic Audio Data Pipeline Plan

## Goal
Build a robust preprocessing and feature pipeline that supports audio of different lengths, avoids speaker leakage, and produces training-ready artifacts for both classical and deep learning models.

## Design Principles
1. Config-driven behavior: all thresholds and strategy choices are configurable.
2. Length-aware processing: short, medium, and long recordings are handled differently.
3. Speaker-safe splitting: train, validation, and test are split by speaker, never by segment.
4. Streaming and batching: process in batches to control memory on long recordings.
5. Reproducibility: every run writes reports and metadata for traceability.

## End-to-End Stages

### 1. Ingestion and Manifesting
Input:
- Raw audio files and labels.

Actions:
- Scan dataset folders and build a manifest with one row per recording.
- Keep recording metadata: speaker_id, path, duration, label source, and quality flags.

Output:
- Master manifest for downstream processing.

### 2. Standardization
Convert all valid recordings to:
- 16 kHz sample rate
- Mono channel
- 16-bit PCM
- WAV format only

Implementation notes:
- Keep standardized files in a dedicated cache directory.
- Skip conversion if standardized version already exists.

### 3. Data Cleaning
Remove or flag:
- Corrupted files
- Extremely noisy files

Apply:
- Leading and trailing silence trimming
- Optional noise reduction

Quality gates:
- Minimum duration after trimming
- Minimum signal energy

### 4. Voice Activity Detection (VAD)
Extract speech-only regions.

Remove:
- Long silence
- Background-only regions

Keep:
- Speech segments with at least 1 second duration

Output:
- Speech region timeline per recording.

### 5. Dynamic Segmentation for Variable-Length Audio
Use a hybrid policy by duration:

1. Short audio (less than window size):
- Keep as single segment.
- Optionally pad for model-specific requirements.

2. Medium audio:
- Fixed windows of 4 to 5 seconds.
- Overlap 25% to 50%.

3. Long audio:
- First apply VAD to isolate speech regions.
- Segment each speech region into fixed windows.
- Process segment windows in chunk batches to avoid memory spikes.

Discard:
- Segments shorter than minimum threshold.

### 6. Label Assignment
If labels are speaker-level:
- Assign speaker label to every segment of that speaker.

If labels are time-based:
- Align each segment using timestamps.
- Store alignment confidence if available.

### 7. Feature Extraction
Choose extraction mode by model family.

For classical or lightweight models:
- MFCC 13 to 40
- Delta and Delta-Delta
- Pitch (F0)
- Energy (RMS)
- Zero Crossing Rate

For deep learning:
- Log-Mel Spectrogram
- Or MFCC stack compatible with your architecture

### 8. Feature Normalization
Apply:
- Mean normalization
- Standard scaling (z-score)

Options:
- Per-feature normalization
- Per-speaker normalization for speaker-level robustness

Rule:
- Fit normalization on training split only, then apply to validation and test.

### 9. Feature Aggregation
Use when model expects fixed-length vectors.

Compute per segment or per recording:
- Mean
- Standard deviation
- Min and max
- Percentiles

Purpose:
- Convert variable-length sequences into fixed-size representations.

### 10. Data Augmentation
Apply on training split only:
- Add Gaussian or background noise
- Time stretch (plus or minus 5% to 10%)
- Pitch shift
- SpecAugment for spectrogram-based models

### 11. Batch Processing and Storage Strategy
To handle different lengths at scale:

1. Chunk batching:
- Process segment groups in fixed chunk batch sizes.

2. Row batching:
- Flush extracted feature rows to disk every N rows.

3. Incremental outputs:
- Write raw features incrementally.
- Fit scaler after raw file is complete.
- Write scaled features and scaler metadata.

Recommended artifact outputs:
- segment_features_raw.csv
- segment_features_scaled.csv
- feature_scaler.json
- preprocess_report.json

### 12. Train, Validation, Test Split
Split by speaker only, never segment.

Recommended ratio:
- 70% train
- 15% validation
- 15% test

Checks:
- No speaker overlap across splits.
- Similar class distribution across splits.

## Practical Implementation Plan

### Phase A: Data Contract and Config
1. Define manifest schema and split schema.
2. Add config entries for sampling, VAD, segmentation, batching, and normalization.
3. Define output locations for intermediate and processed artifacts.

### Phase B: Preprocessing Core
1. Implement standardization and cleaning.
2. Implement VAD and dynamic segmentation policy.
3. Implement chunk-batch processing and row-batch disk flushing.

### Phase C: Feature Pipeline
1. Implement extraction for selected feature sets.
2. Add normalization and scaler persistence.
3. Add optional aggregation path for classical models.

### Phase D: Training Readiness
1. Build speaker-level splits from manifests.
2. Ensure split leakage checks are automatic.
3. Add optional augmentation for training split only.

### Phase E: Validation and Monitoring
1. Write preprocess reports with counts and drop reasons.
2. Track segment length distribution and class balance after segmentation.
3. Add smoke tests for short, medium, and long audio samples.

## Dynamic Behavior Summary
The pipeline is dynamic because it adapts segmentation strategy to recording length, uses VAD-guided speech-only chunking, and processes features in configurable batches. This keeps quality high for short clips, preserves context for long recordings, and maintains stable memory usage during large-scale preprocessing.
