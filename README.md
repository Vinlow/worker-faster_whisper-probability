# Faster Whisper RunPod Worker (with per-word probability)

Fork of [runpod-workers/worker-faster_whisper](https://github.com/runpod-workers/worker-faster_whisper) with **per-word confidence scores** and optimized for video editing pipelines.

## What changed from upstream

### Per-word probability output

When `word_timestamps` is enabled, each word now includes its `probability` (0-1 confidence score from faster-whisper). The upstream worker stripped this field.

```json
{
  "word_timestamps": [
    { "word": "Hello", "start": 0.0, "end": 0.5, "probability": 0.97 },
    { "word": "world", "start": 0.5, "end": 1.0, "probability": 0.84 }
  ]
}
```

This enables downstream consumers to:
- Score transcript quality per-word (not just per-segment)
- Identify garbled or uncertain transcriptions
- Make smarter decisions when picking between retakes of the same content

### Dependency upgrades

| Package | Upstream | This fork |
|---|---|---|
| faster-whisper | 1.1.0 | **1.2.1** |
| runpod SDK | 1.7.9 | **1.8.2** |

Faster-whisper 1.2.1 brings Silero-VAD V6, `distil-large-v3.5` model support, better timestamp handling, and RAM usage fixes.

### Smaller Docker image

Only pre-downloads `large-v3`, `distil-large-v3.5`, and `turbo` instead of all 10 models. Other models remain available on-demand (downloaded on first request). Cuts image size by ~15-20 GB.

---

## Models

Pre-downloaded (instant cold start):
- **large-v3** — highest quality, ~3x realtime on A40
- **distil-large-v3.5** — near large-v3 quality, ~5x faster
- **turbo** — fastest, good for draft transcriptions

Available on-demand (downloaded on first use):
- tiny, base, small, medium, large-v1, large-v2, distil-large-v2, distil-large-v3

## Input

| Input                               | Type  | Description                                                                                                                                                            |
| ----------------------------------- | ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `audio`                             | Path  | URL to audio file                                                                                                                                                      |
| `audio_base64`                      | str   | Base64-encoded audio file                                                                                                                                              |
| `model`                             | str   | Whisper model. Default: `"base"` |
| `transcription`                     | str   | Output format: `"plain_text"`, `"formatted_text"`, `"srt"`, `"vtt"`. Default: `"plain_text"` |
| `translate`                         | bool  | Translate to English. Default: `false` |
| `translation`                       | str   | Translation output format. Default: `"plain_text"` |
| `language`                          | str   | Language code, or `null` for auto-detection. Default: `null` |
| `temperature`                       | float | Sampling temperature. Default: `0` |
| `best_of`                           | int   | Candidates when sampling with non-zero temperature. Default: `5` |
| `beam_size`                         | int   | Beam search width (when temperature is 0). Default: `5` |
| `patience`                          | float | Beam decoding patience. Default: `1.0` |
| `length_penalty`                    | float | Token length penalty coefficient. Default: `0` |
| `suppress_tokens`                   | str   | Comma-separated token IDs to suppress. Default: `"-1"` |
| `initial_prompt`                    | str   | Prompt text for the first window. Default: `null` |
| `condition_on_previous_text`        | bool  | Feed previous output as prompt for next window. Default: `true` |
| `temperature_increment_on_fallback` | float | Temperature increment on decoding failure. Default: `0.2` |
| `compression_ratio_threshold`       | float | Compression ratio threshold for failure detection. Default: `2.4` |
| `logprob_threshold`                 | float | Average log probability threshold. Default: `-1.0` |
| `no_speech_threshold`               | float | No-speech probability threshold. Default: `0.6` |
| `enable_vad`                        | bool  | Enable Silero VAD to filter non-speech. Default: `false` |
| `word_timestamps`                   | bool  | Include per-word timestamps and probability. Default: `false` |

## Output

### Segments

Always returned. Each segment includes quality metrics:

```json
{
  "segments": [
    {
      "id": 0,
      "start": 0.0,
      "end": 5.2,
      "text": " Four score and seven years ago...",
      "avg_logprob": -0.12,
      "compression_ratio": 1.68,
      "no_speech_prob": 0.05
    }
  ]
}
```

### Word timestamps (when `word_timestamps: true`)

```json
{
  "word_timestamps": [
    { "word": "Four", "start": 0.0, "end": 0.3, "probability": 0.98 },
    { "word": "score", "start": 0.3, "end": 0.6, "probability": 0.95 }
  ]
}
```

**`probability`** is the model's confidence in each word (0.0 - 1.0). Higher = more confident. Useful for:
- Detecting transcription errors (low probability = likely wrong)
- Comparing retake quality in video editing
- Confidence-weighted subtitle rendering

## Example

```json
{
  "input": {
    "audio": "https://github.com/runpod-workers/sample-inputs/raw/main/audio/gettysburg.wav",
    "model": "large-v3",
    "word_timestamps": true,
    "enable_vad": true
  }
}
```
