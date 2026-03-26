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

4. **Join segments with re-encoding** using trim/atrim filters in a single ffmpeg pass. Do NOT use stream copy (`-c copy`) — stream copy snaps to keyframes, which causes cumulative timestamp drift in the segment map (critical for multi-angle) and audio blips at segment boundaries from overlapping content.
   - Build a `filter_complex` with `trim`/`atrim` + `setpts`/`asetpts` for each segment, then concat:
     ```
     [0:v]trim=start=S1:end=E1,setpts=PTS-STARTPTS[v0];
     [0:a]atrim=start=S1:end=E1,asetpts=PTS-STARTPTS[a0];
     [0:v]trim=start=S2:end=E2,setpts=PTS-STARTPTS[v1];
     [0:a]atrim=start=S2:end=E2,asetpts=PTS-STARTPTS[a1];
     ...
     [v0][a0][v1][a1]...concat=n=N:v=1:a=1[outv][outa]
     ```
   - Write the filter to a PID-unique temp file (e.g., `/tmp/silence_filter_{pid}.txt`)
   - Render: `ffmpeg -y -i <input> -filter_complex_script <filter_file> -map "[outv]" -map "[outa]" -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 192k <output>`
   - Delete the temp filter file after rendering

5. **Write segment map** JSON file alongside the trimmed output. The segment map records which time ranges from the input were kept, enabling downstream timestamp mapping (e.g., for produce-zoom to map trimmed timestamps back to original file timestamps).

   Output: `<name>_segment_map.json` alongside `<name>_trimmed.mp4`

   Format:
   ```json
   [
     {"trimmed_start": 0.000, "trimmed_end": 5.200, "original_start": 0.000, "original_end": 5.200},
     {"trimmed_start": 5.200, "trimmed_end": 10.800, "original_start": 6.500, "original_end": 12.100}
   ]
   ```

   Each entry maps a contiguous range in the trimmed output back to its original position in the input file. The `trimmed_start` of each entry equals the `trimmed_end` of the previous entry (they are contiguous in the output). The `original_start`/`original_end` values are the actual timestamps from the input file.

6. **Output** the trimmed file as `<base>_trimmed.<ext>` in the same directory, plus the segment map as `<base>_segment_map.json`. To compute `<base>`: take the input filename without extension, then strip any `_synced` suffix (so `mainvideo_synced.mp4` → base = `mainvideo`, outputs = `mainvideo_trimmed.mp4` + `mainvideo_segment_map.json`). This ensures consistent naming whether or not the pipeline includes a sync step.

### Important notes

- The video stream index may not be `0:0` -- check `ffprobe` output. Audio and video stream indices vary per file.
- Always use `[0:v]` and `[0:a]` in filter references to let ffmpeg pick the right streams.
- Report the original duration, trimmed duration, and time saved.
- **Segment map**: The segment map JSON is always written, even for single-camera workflows. It is cheap to produce and keeps the pipeline uniform. Downstream steps (like produce-zoom in multi-angle mode) use it to map trimmed timestamps back to original (synced primary) timestamps for extracting secondary footage at the correct position.

### Example Python script pattern

```python
import re, subprocess, json, os

video = "<input_path>"
# Derive base name: strip extension, then strip _synced suffix if present
base_name = os.path.splitext(video)[0]
if base_name.endswith("_synced"):
    base_name = base_name[:-len("_synced")]
ext = os.path.splitext(video)[1]
output = f"{base_name}_trimmed{ext}"
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

# Build filter_complex with trim/atrim for frame-accurate cuts
filter_parts = []
for i, (s, e) in enumerate(segments):
    filter_parts.append(f"[0:v]trim=start={s:.6f}:end={e:.6f},setpts=PTS-STARTPTS[v{i}];")
    filter_parts.append(f"[0:a]atrim=start={s:.6f}:end={e:.6f},asetpts=PTS-STARTPTS[a{i}];")

concat_inputs = "".join(f"[v{i}][a{i}]" for i in range(len(segments)))
filter_parts.append(f"{concat_inputs}concat=n={len(segments)}:v=1:a=1[outv][outa]")

filter_file = f"/tmp/silence_filter_{os.getpid()}.txt"
with open(filter_file, "w") as f:
    f.write("\n".join(filter_parts))

# Render with re-encoding (frame-accurate, no keyframe drift)
subprocess.run([
    "ffmpeg", "-y", "-i", video,
    "-filter_complex_script", filter_file,
    "-map", "[outv]", "-map", "[outa]",
    "-c:v", "libx264", "-preset", "fast", "-crf", "18",
    "-c:a", "aac", "-b:a", "192k",
    output
], check=True, capture_output=True)

os.remove(filter_file)

# Build and write segment map
segment_map = []
trimmed_pos = 0.0
for orig_start, orig_end in segments:
    seg_duration = orig_end - orig_start
    segment_map.append({
        "trimmed_start": round(trimmed_pos, 6),
        "trimmed_end": round(trimmed_pos + seg_duration, 6),
        "original_start": round(orig_start, 6),
        "original_end": round(orig_end, 6)
    })
    trimmed_pos += seg_duration

# Write segment map JSON
# Strip _synced suffix so multi-angle naming stays consistent
base_name = os.path.splitext(video)[0]
if base_name.endswith("_synced"):
    base_name = base_name[:-len("_synced")]
segment_map_path = f"{base_name}_segment_map.json"
with open(segment_map_path, "w") as f:
    json.dump(segment_map, f, indent=2)
```
