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

#### Step 4: Calculate common overlap

All feeds must be trimmed to the time range where all cameras were recording simultaneously.

```python
def calculate_overlap(primary_duration, secondaries_info):
    """Calculate the common overlap across all feeds.

    secondaries_info: list of (offset, duration) tuples for each secondary.

    Returns (primary_start, primary_end, secondary_starts) where:
      - primary_start/end are times in primary's timeline
      - secondary_starts[i] is the corresponding start time in secondary i's timeline
    """
    # In primary's timeline:
    #   Primary spans [0, primary_duration]
    #   Secondary i spans [offset_i, offset_i + duration_i]
    common_start = 0.0
    common_end = primary_duration

    for offset, duration in secondaries_info:
        common_start = max(common_start, offset)
        common_end = min(common_end, offset + duration)

    if common_end <= common_start:
        raise ValueError("No overlapping time range found — feeds may not be from the same recording")

    # Convert common range to each secondary's timeline
    secondary_starts = []
    for offset, duration in secondaries_info:
        sec_start = common_start - offset
        secondary_starts.append(sec_start)

    overlap_duration = common_end - common_start
    return common_start, common_end, secondary_starts, overlap_duration
```

#### Step 5: Trim all feeds to common overlap

Use stream copy (no re-encoding) for speed. Strip audio from secondaries with `-an`.

```bash
# Trim primary — keep video and audio
ffmpeg -y -ss <common_start> -to <common_end> -i <primary> \
  -c copy -avoid_negative_ts make_zero \
  <primary_basename>_synced.mp4

# Trim each secondary — keep video only, strip audio
ffmpeg -y -ss <secondary_start> -to <secondary_end> -i <secondary> \
  -c:v copy -an -avoid_negative_ts make_zero \
  <secondary_basename>_synced.mp4
```

#### Step 6: Verify sync

Confirm the synced files have matching durations (within one frame).

```python
primary_synced_dur = get_duration(primary_synced_path)
for sec_synced_path in secondary_synced_paths:
    sec_synced_dur = get_duration(sec_synced_path)
    diff = abs(primary_synced_dur - sec_synced_dur)
    fps = 30  # assume 30fps; could probe actual fps
    frame_dur = 1.0 / fps
    if diff > frame_dur:
        print(f"WARNING: Duration mismatch of {diff:.3f}s between primary and {sec_synced_path}")
    else:
        print(f"Sync verified: durations match within 1 frame ({diff*1000:.1f}ms)")
```

#### Step 7: Clean up temp files

Delete the extracted WAV files:
```bash
rm -f /tmp/sync_primary_$PID.wav /tmp/sync_sec*_$PID.wav
```

### Output

- `<primary_basename>_synced.mp4` — primary video trimmed to common overlap, audio preserved
- `<secondary_basename>_synced.mp4` — each secondary trimmed to common overlap, audio stripped (no audio track)

All output files go in the same directory as the input files.

### Report

Print:
- Offset found for each secondary (in seconds and in frames at the video's fps)
- Correlation confidence for each secondary
- Original duration of each feed
- Common overlap duration
- Time trimmed from the start and end of each feed
- Output file paths

### Important notes

- **Audio required for sync**: All cameras must have captured audible room audio for cross-correlation to work. If a secondary has no audio track, this method won't work — the user would need to provide the offset manually (e.g., from a clap/slate).
- **Same room, same event**: This assumes all cameras were recording the same audio event in the same room. It will not work for unrelated recordings.
- **30-second correlation window**: Uses a 30-second chunk for speed. For recordings under 60 seconds, the full secondary audio is used instead.
- **Stream copy**: Trimming uses `-c copy` / `-c:v copy` to avoid re-encoding. This means the trim points snap to the nearest keyframe. For talking-head video this is fine — the error is typically under 0.5 seconds and consistent across all feeds.
- **Secondary audio is stripped**: The `-an` flag removes the audio track entirely from secondary outputs. Only the primary's audio track is kept for the final edit.
- **Output is always .mp4**: Regardless of input format, synced outputs are .mp4 for pipeline compatibility.
- **Frame rate mismatch**: If cameras recorded at different frame rates, ffmpeg will preserve each file's native frame rate during trim. The produce-zoom step later handles any frame rate normalization during re-encoding.
