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

4. **Render trimmed video** using ffmpeg's trim/atrim filters with concat:
   - Build a filter_complex with `trim`/`setpts` for video and `atrim`/`asetpts` for audio per segment
   - Concat inputs must be interleaved: `[v0][a0][v1][a1]...concat=n=N:v=1:a=1`
   - Write filter to a temp file and use `-/filter_complex <file>` (ffmpeg 8.x) or `-filter_complex_script <file>` (older ffmpeg)
   - Encode with `-c:v libx264 -preset fast -crf 18 -c:a aac -b:a 192k`

5. **Output** the trimmed file as `<original_name>_trimmed.<ext>` in the same directory

### Important notes

- The video stream index may not be `0:0` -- check `ffprobe` output. Audio and video stream indices vary per file.
- Always use `[0:v]` and `[0:a]` in filter references to let ffmpeg pick the right streams.
- Report the original duration, trimmed duration, and time saved.

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

# Build filter_complex
parts = []
for i, (s, e) in enumerate(segments):
    parts.append(f"[0:v]trim=start={s:.6f}:end={e:.6f},setpts=PTS-STARTPTS[v{i}];")
    parts.append(f"[0:a]atrim=start={s:.6f}:end={e:.6f},asetpts=PTS-STARTPTS[a{i}];")
concat_inputs = "".join(f"[v{i}][a{i}]" for i in range(len(segments)))
parts.append(f"{concat_inputs}concat=n={len(segments)}:v=1:a=1[outv][outa]")

filter_complex = "\n".join(parts)
import os
filter_tmp = f"/tmp/silence_filter_{os.getpid()}.txt"
with open(filter_tmp, "w") as f:
    f.write(filter_complex)

cmd = ["ffmpeg", "-y", "-i", video, "-/filter_complex", filter_tmp,
       "-map", "[outv]", "-map", "[outa]",
       "-c:v", "libx264", "-preset", "fast", "-crf", "18",
       "-c:a", "aac", "-b:a", "192k", output]
subprocess.run(cmd, check=True)
os.remove(filter_tmp)
```
