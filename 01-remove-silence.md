---
name: remove-silence
description: Remove silence from video files, trimming dead air while preserving natural pauses
user_invocable: true
metadata:
  tags: video, audio, ffmpeg, silence, editing
---

## When to use

Use this skill when the user wants to remove silences, dead air, or long pauses from a video file. Triggered by requests like "remove silence," "trim dead air," "cut pauses," or "remove-silence."

## How to use

### Prerequisites
- `ffmpeg` must be installed (check with `which ffmpeg`)
- Input must be a video file (mp4, mov, mkv, webm, etc.)

### Parameters
The user may optionally specify:
- **input**: Path to the video file (if not provided, ask)
- **secondaries**: Paths to secondary video files (synced angles, from `/sync-feeds`). When provided, the same silence cuts are applied to all secondary feeds using the primary's audio for detection. Secondary outputs are video-only (no audio).
- **silence_threshold**: Minimum silence duration to detect (default: 0.5 seconds)
- **pause_duration**: How much natural pause to leave behind (default: 0.3 seconds)
- **noise_level**: dB threshold for silence detection (default: -30dB)

### Process

1. **Detect silences** using ffmpeg's `silencedetect` audio filter:
   ```
   ffmpeg -i <input> -af silencedetect=noise=<noise_level>:d=<silence_threshold> -f null -
   ```

2. **Parse silence regions** from stderr output (silence_start / silence_end pairs)

3. **Build non-silent segments** by:
   - For each silence, keep `pause_duration / 2` seconds on each side
   - Merge overlapping or adjacent segments

4. **Extract segments with stream copy** (no re-encoding):
   - For each non-silent segment, extract to a temp file using `ffmpeg -ss <start> -to <end> -i <input> -c copy -avoid_negative_ts make_zero /tmp/seg_NNN.mp4`
   - Write a concat demuxer list file: `file '/tmp/seg_000.mp4'\nfile '/tmp/seg_001.mp4'\n...`
   - Concatenate all segments: `ffmpeg -f concat -safe 0 -i <list_file> -c copy <output>`
   - Clean up temp segment files after concatenation

5. **Apply same cuts to secondary feeds** (only when secondaries are provided):
   - Use the exact same segment time ranges from step 3 (detected on the primary's audio)
   - For each secondary file, extract segments with: `ffmpeg -ss <start> -to <end> -i <secondary> -c:v copy -an -avoid_negative_ts make_zero /tmp/sec_seg_NNN.mp4`
   - Note: `-an` strips audio since secondaries are video-only (audio was stripped by `/sync-feeds`)
   - Concatenate each secondary's segments separately: `ffmpeg -f concat -safe 0 -i <list_file> -c:v copy -an <secondary_output>`
   - Output as `<secondary_name>_trimmed.mp4`

6. **Output** the trimmed file as `<original_name>_trimmed.<ext>` in the same directory. If secondaries were provided, their trimmed versions are also in the same directory.

### Important notes

- The video stream index may not be `0:0` -- check `ffprobe` output. Audio and video stream indices vary per file.
- Always use `[0:v]` and `[0:a]` in filter references to let ffmpeg pick the right streams.
- Report the original duration, trimmed duration, and time saved.
- **Multi-angle**: When secondaries are provided, silence detection runs only on the primary (it has the good audio). The same time segments are cut from all feeds, keeping them in sync. Secondary extractions use `-c:v copy -an` since they have no audio track.

### Example Python script pattern

```python
import re, subprocess

video = "<input_path>"
output = "<output_path>"
pause = 0.3
noise = "-30dB"
min_silence = 0.5

# Get duration
dur_out = subprocess.run(["ffprobe", "-v", "error", "-show_entries", "format=duration",
    "-of", "default=noprint_wrappers=1:nokey=1", video], capture_output=True, text=True)
duration = float(dur_out.stdout.strip())

# Detect silences
result = subprocess.run(["ffmpeg", "-i", video, "-af",
    f"silencedetect=noise={noise}:d={min_silence}", "-f", "null", "-"],
    capture_output=True, text=True)

silences = []
starts = re.findall(r"silence_start: ([\d.]+)", result.stderr)
ends = re.findall(r"silence_end: ([\d.]+)", result.stderr)
for s, e in zip(starts, ends):
    silences.append((float(s), float(e)))

# Build segments
half_pause = pause / 2
segments = []
pos = 0.0
for s_start, s_end in silences:
    seg_end = s_start + half_pause
    if seg_end > pos:
        segments.append((pos, seg_end))
    pos = s_end - half_pause
if pos < duration:
    segments.append((pos, duration))

# Merge overlapping
merged = [segments[0]]
for s, e in segments[1:]:
    if s <= merged[-1][1]:
        merged[-1] = (merged[-1][0], max(merged[-1][1], e))
    else:
        merged.append((s, e))
segments = merged

# Extract segments with stream copy (no re-encoding)
import os, tempfile
tmpdir = tempfile.mkdtemp(prefix="silence_")
seg_files = []

for i, (s, e) in enumerate(segments):
    seg_path = os.path.join(tmpdir, f"seg_{i:04d}.mp4")
    seg_files.append(seg_path)
    subprocess.run([
        "ffmpeg", "-y", "-ss", f"{s:.6f}", "-to", f"{e:.6f}",
        "-i", video, "-c", "copy", "-avoid_negative_ts", "make_zero",
        seg_path
    ], check=True, capture_output=True)

# Write concat list
concat_list = os.path.join(tmpdir, "concat.txt")
with open(concat_list, "w") as f:
    for seg_path in seg_files:
        f.write(f"file '{seg_path}'\n")

# Concatenate with stream copy
subprocess.run([
    "ffmpeg", "-y", "-f", "concat", "-safe", "0",
    "-i", concat_list, "-c", "copy", output
], check=True)

# Clean up temp segments
import shutil
shutil.rmtree(tmpdir)
```
