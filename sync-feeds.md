---
name: sync-feeds
description: Sync multiple camera feeds by audio cross-correlation for multi-angle video editing
user_invocable: true
metadata:
  tags: video, sync, multi-camera, audio, cross-correlation, multi-angle
---

## When to use

Use this skill when the user has multiple camera angles of the same recording and needs to sync them before editing. Triggered by `/sync-feeds` or requests like "sync these videos," "align the cameras," or "sync the angles."

## How to use

### Prerequisites
- `ffmpeg` must be installed
- `scipy` and `numpy` must be installed (`pip3 install scipy numpy`)

### Parameters
- **primary**: Path to the primary video file (the one with the quality microphone). This is always the first argument.
- **secondaries**: Paths to 1-2 secondary video files (other camera angles). These follow the primary.

If the user doesn't specify which is primary, ask. The primary is the one with the good audio — it provides the audio track for the final output.

Example invocations:
- `/sync-feeds main_camera.mp4 side_angle.mp4`
- `/sync-feeds primary.mp4 angle2.mp4 angle3.mp4`

### Process

#### Step 1: Extract audio from all feeds

Extract mono 16kHz WAV from each file for cross-correlation. Use PID-unique temp filenames to avoid collisions.

```bash
ffmpeg -y -i <primary> -ac 1 -ar 16000 -f wav /tmp/sync_primary_$PID.wav
ffmpeg -y -i <secondary1> -ac 1 -ar 16000 -f wav /tmp/sync_sec1_$PID.wav
# repeat for secondary2 if present
```

#### Step 2: Get durations of all feeds

```python
import subprocess

def get_duration(path):
    result = subprocess.run(
        ["ffprobe", "-v", "error", "-show_entries", "format=duration",
         "-of", "default=noprint_wrappers=1:nokey=1", path],
        capture_output=True, text=True
    )
    return float(result.stdout.strip())
```

#### Step 3: Cross-correlate to find offset

For each secondary, find its time offset relative to the primary's timeline.

```python
import numpy as np
from scipy.io import wavfile
from scipy.signal import fftconvolve

def find_offset(primary_wav_path, secondary_wav_path):
    """Find the offset of secondary relative to primary's timeline.

    Returns (offset_seconds, confidence).

    offset_seconds: the time in primary's timeline where secondary's t=0 aligns.
      Positive = secondary started recording after primary.
      Negative = secondary started recording before primary.

    confidence: normalized correlation peak (0-1). Above 0.3 is typically a good match.
    """
    sr_p, primary_audio = wavfile.read(primary_wav_path)
    sr_s, secondary_audio = wavfile.read(secondary_wav_path)
    assert sr_p == sr_s, "Sample rates must match"
    sample_rate = sr_p

    # Use a 30-second chunk from the middle of the secondary
    chunk_duration = 30  # seconds
    chunk_samples = chunk_duration * sample_rate

    mid = len(secondary_audio) // 2
    half_chunk = min(chunk_samples // 2, mid)
    chunk = secondary_audio[mid - half_chunk : mid + half_chunk].astype(np.float64)
    primary_f = primary_audio.astype(np.float64)

    # Normalize to [-1, 1]
    chunk_max = np.max(np.abs(chunk))
    primary_max = np.max(np.abs(primary_f))
    if chunk_max > 0:
        chunk /= chunk_max
    if primary_max > 0:
        primary_f /= primary_max

    # Cross-correlate using FFT (fast for large arrays)
    correlation = fftconvolve(primary_f, chunk[::-1], mode='full')

    # Find peak
    peak_idx = np.argmax(np.abs(correlation))
    peak_value = np.abs(correlation[peak_idx])

    # Confidence: normalized correlation
    confidence = peak_value / np.sqrt(np.sum(chunk**2) * np.sum(primary_f**2))

    # Convert peak position to time offset
    # In 'full' mode, at peak_idx the chunk's start aligns at position:
    #   (peak_idx - len(chunk) + 1) in primary's sample space
    chunk_start_in_primary_samples = peak_idx - len(chunk) + 1
    chunk_start_in_primary_seconds = chunk_start_in_primary_samples / sample_rate

    # The chunk was taken from secondary starting at sample (mid - half_chunk)
    chunk_origin_in_secondary = (mid - half_chunk) / sample_rate

    # Secondary's t=0 in primary's timeline
    offset = chunk_start_in_primary_seconds - chunk_origin_in_secondary

    return offset, confidence
```

**Confidence threshold**: If confidence is below 0.15, warn the user that the sync may be unreliable. Below 0.05, stop and ask the user to verify the files contain overlapping audio.

#### Step 4: Calculate common overlap and trim primary only

Calculate the time range where all cameras were recording simultaneously. Trim only the **primary** to this range (stream copy). Secondaries are **not trimmed** — they stay as original files.

```python
def calculate_overlap(primary_duration, secondaries_info):
    """Calculate the common overlap across all feeds.

    secondaries_info: list of (offset, duration) tuples for each secondary.

    Returns (primary_start, primary_end, overlap_duration).
    """
    common_start = 0.0
    common_end = primary_duration

    for offset, duration in secondaries_info:
        common_start = max(common_start, offset)
        common_end = min(common_end, offset + duration)

    if common_end <= common_start:
        raise ValueError("No overlapping time range found — feeds may not be from the same recording")

    return common_start, common_end, common_end - common_start
```

Trim the primary to the overlap range:

```bash
# Trim primary only — keep video and audio
ffmpeg -y -ss <common_start> -to <common_end> -i <primary> \
  -c copy -avoid_negative_ts make_zero \
  <primary_basename>_synced.mp4
```

**Do NOT trim the secondaries.** They stay as original files. The produce-zoom step will extract from them at the correct timestamps during re-encoding, which gives frame-accurate alignment without keyframe snapping issues.

#### Step 5: Write sync manifest

Write a JSON manifest that records everything downstream steps need to find secondary footage for any moment in the synced primary's timeline.

```python
import json

manifest = {
    "primary_original": str(primary_path),
    "primary_synced": str(synced_primary_path),
    "overlap": {
        "primary_start": common_start,
        "primary_end": common_end,
        "duration": overlap_duration
    },
    "secondaries": []
}

for i, (sec_path, offset, confidence) in enumerate(secondary_results):
    manifest["secondaries"].append({
        "file": str(sec_path),
        "sync_offset": offset,
        "confidence": confidence,
        "index": i + 1  # secondary_1, secondary_2
    })

manifest_path = primary_path.replace(".mp4", "_sync_manifest.json")
with open(manifest_path, "w") as f:
    json.dump(manifest, f, indent=2)
```

The manifest enables timestamp mapping. For any time `t` in the synced primary's timeline, the corresponding time in a secondary's original file is:

```python
secondary_time = t + manifest["overlap"]["primary_start"] - secondary["sync_offset"]
```

This formula works because:
- `t` is relative to the synced primary (which starts at `overlap.primary_start` in the original primary's timeline)
- `t + primary_start` converts to original primary time
- Subtracting `sync_offset` converts from primary time to secondary time

#### Step 6: Clean up temp files

Delete the extracted WAV files:
```bash
rm -f /tmp/sync_primary_$PID.wav /tmp/sync_sec*_$PID.wav
```

### Output

- `<primary_basename>_synced.mp4` — primary video trimmed to common overlap, audio preserved
- `<primary_basename>_sync_manifest.json` — manifest with offsets and secondary file paths
- Secondary files are **not modified** — they stay as-is in their original location

### Report

Print:
- Offset found for each secondary (in seconds and in frames at the video's fps)
- Correlation confidence for each secondary
- Original duration of each feed
- Common overlap duration
- Time trimmed from the start and end of the primary
- Timestamp mapping formula for verification
- Manifest file path

### Important notes

- **Audio required for sync**: All cameras must have captured audible room audio for cross-correlation to work. If a secondary has no audio track, this method won't work — the user would need to provide the offset manually (e.g., from a clap/slate).
- **Same room, same event**: This assumes all cameras were recording the same audio event in the same room. It will not work for unrelated recordings.
- **30-second correlation window**: Uses a 30-second chunk for speed. For recordings under 60 seconds, the full secondary audio is used instead.
- **Only the primary is trimmed.** The primary is trimmed with stream copy to the overlap range. Keyframe snapping on this trim is fine — the synced primary becomes the reference timeline. All downstream timestamps are relative to it.
- **Secondaries stay untouched.** No trimming, no re-encoding, no stream copy. The produce-zoom step extracts from the original secondary files at frame-accurate timestamps during its re-encoding pass. This avoids keyframe alignment issues entirely.
- **The manifest is the sync data.** Downstream steps (produce-zoom) read the manifest to find secondary files and compute timestamps. The manifest travels with the synced primary through the pipeline.
