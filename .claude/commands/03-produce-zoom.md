---
name: produce-zoom
description: Apply dynamic zoom sections from JSON to a video using face detection and ffmpeg cropping
user_invocable: true
metadata:
  tags: video, zoom, crop, face-detection, ffmpeg, editing
---

## When to use

Use this skill when the user has a sections JSON file (from `/label-zoom-sections`) and wants to render the zoomed/cropped video. Triggered by requests like "make the zoom video," "apply zoom sections," "render the zoomed video," or "make-zoom-video."

## How to use

### Prerequisites
- `ffmpeg` must be installed
- `opencv-python-headless` must be installed (`pip3 install opencv-python-headless`)
- A sections JSON file (produced by `/label-zoom-sections`)
- The trimmed video file referenced in the JSON

### Parameters
The user may optionally specify:
- **input**: Path to the primary video file (if not provided, infer from JSON metadata or ask)
- **sections_json**: Path to the sections JSON file (if not provided, look for `<input_basename>_sections.json`). If the JSON contains `secondary_sources` in metadata and `source` fields on sections (from `/swap-angles`), multi-angle mode is enabled automatically.
- **sync_manifest**: Path to the sync manifest JSON (from `/sync-feeds`). Required for multi-angle mode. If not provided, first check the sections JSON metadata for `sync_manifest`; otherwise derive from the input basename by stripping `_trimmed` and looking for `<original_name>_sync_manifest.json` (e.g., input `mainvideo_trimmed.mp4` → look for `mainvideo_sync_manifest.json`).
- **segment_map**: Path to the segment map JSON (from `/remove-silence`). Required for multi-angle mode. If not provided, derive from the input basename by stripping `_trimmed` and looking for `<original_name>_segment_map.json` (e.g., input `mainvideo_trimmed.mp4` → look for `mainvideo_segment_map.json`).
- **output**: Path for the output file (default: strip `_trimmed` from the input basename, then append `_zoomed.mp4` — e.g., `mainvideo_trimmed.mp4` → `mainvideo_zoomed.mp4`)
- **resolution**: `-HD` (default), `-4K`, or `-portrait` (1080x1920 vertical 9:16)

### Process

#### Step 1: Load sections JSON, video metadata, and multi-angle data

Read the sections JSON file. It has this structure:

```json
{
  "metadata": {
    "source": "testvideo_trimmed.mp4",
    "duration": 203.6,
    "zoom_levels": {
      "normal": 1.0,
      "emphasis": 1.25,
      "critical": 1.6
    }
  },
  "sections": [
    { "start": 0.000, "end": 6.360, "label": "normal", "text": "..." }
  ]
}
```

Use the `zoom_levels` from the JSON metadata. These map label names to zoom multipliers.

**Multi-angle detection**: If `metadata.secondary_sources` exists and sections have `"source"` fields, this is a multi-angle edit. Load the secondary video file paths from metadata. Build a source map:
- `"primary"` -> input file (ffmpeg input index 0)
- `"secondary_1"` -> first secondary file (ffmpeg input index 1)
- `"secondary_2"` -> second secondary file (ffmpeg input index 2)

##### Loading the sync manifest and segment map (multi-angle)

When multi-angle mode is detected, load the sync manifest and segment map:

```python
import json

# Load sync manifest
with open(manifest_path) as f:
    manifest = json.load(f)

# Load segment map
with open(segment_map_path) as f:
    segment_map = json.load(f)
```

The sync manifest provides:
- `manifest["overlap"]["primary_start"]` — where the synced primary starts in the original primary's timeline
- `manifest["secondaries"][i]["sync_offset"]` — each secondary's offset in the original primary's timeline
- `manifest["secondaries"][i]["file"]` — the path to each secondary's original (untrimmed) file

The segment map provides:
- A list of entries mapping trimmed timestamps back to original (synced primary) timestamps
- Each entry has `trimmed_start`, `trimmed_end`, `original_start`, `original_end`

Together these enable mapping any timestamp in the trimmed primary to the correct position in a secondary's original file.

#### Helper functions: timestamp mapping

```python
def trimmed_to_original(trimmed_time, segment_map):
    """Map a timestamp in the trimmed primary to its position in the synced primary."""
    for seg in segment_map:
        if seg["trimmed_start"] <= trimmed_time <= seg["trimmed_end"]:
            offset_within = trimmed_time - seg["trimmed_start"]
            return seg["original_start"] + offset_within
    return trimmed_time  # fallback

def trimmed_to_secondary(trimmed_time, segment_map, manifest, secondary_index=0):
    """Map a timestamp in the trimmed primary to the corresponding timestamp
    in a secondary's original (untrimmed) file.

    The chain is:
      trimmed time -> synced primary time -> original primary time -> secondary time

    Since the synced primary IS the original primary trimmed to the overlap start,
    adding primary_start converts synced->original. Then subtracting sync_offset
    converts original primary time -> secondary time.
    """
    original_time = trimmed_to_original(trimmed_time, segment_map)
    sec = manifest["secondaries"][secondary_index]
    return original_time + manifest["overlap"]["primary_start"] - sec["sync_offset"]
```

#### Step 2: Detect face position (primary only)

Use OpenCV's Haar cascade face detector to find the average face center across sampled frames on the **primary** video. Face detection is only needed for the primary — secondary angle sections are rendered full-frame with no zoom or face-centering.

```python
import cv2

cap = cv2.VideoCapture(video)
width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))

face_cascade = cv2.CascadeClassifier(
    cv2.data.haarcascades + 'haarcascade_frontalface_default.xml'
)

face_centers = []
sample_points = [int(total_frames * i / 10) for i in range(10)]

for frame_idx in sample_points:
    cap.set(cv2.CAP_PROP_POS_FRAMES, frame_idx)
    ret, frame = cap.read()
    if not ret:
        continue
    scale = 0.25
    small = cv2.resize(frame, None, fx=scale, fy=scale)
    gray = cv2.cvtColor(small, cv2.COLOR_BGR2GRAY)
    faces = face_cascade.detectMultiScale(gray, 1.1, 5, minSize=(30, 30))
    if len(faces) > 0:
        faces = sorted(faces, key=lambda f: f[2]*f[3], reverse=True)
        x, y, w, h = faces[0]
        cx = int((x + w/2) / scale)
        cy = int((y + h/2) / scale)
        face_centers.append((cx, cy))

cap.release()

# Average face position, or frame center as fallback
if face_centers:
    face_cx = int(sum(c[0] for c in face_centers) / len(face_centers))
    face_cy = int(sum(c[1] for c in face_centers) / len(face_centers))
else:
    face_cx, face_cy = width // 2, height // 2
```

**Outlier filtering**: If a detected face center is more than 30% of frame width/height away from the median, discard it as a false positive before averaging.

#### Step 3: Build ffmpeg filter

For each section, calculate the crop region based on the zoom level and resolution mode.

##### Landscape mode (default — `-HD` or `-4K`)

```python
zoom = zoom_levels[section["label"]]

# Crop dimensions (what portion of the frame to keep)
cw = int(width / zoom)
ch = int(height / zoom)
cw = cw - (cw % 2)  # must be even
ch = ch - (ch % 2)

# Position crop centered on face, clamped to frame bounds
cx = max(0, min(face_cx - cw // 2, width - cw))
cy = max(0, min(face_cy - ch // 2, height - ch))
```

##### Portrait mode (`-portrait`)

Portrait crops a 9:16 region from the 16:9 source. Since the source is wider than tall, the crop is always narrower than the source width. The zoom levels control how much of the source **height** is used, which determines the framing:

- **normal** (full frame): use 100% of source height -> no crop, establishes the scene
- **emphasis** (head-and-shoulders): use ~70% of source height -> clear crop, noticeably tighter
- **critical** (tight face): use ~45% of source height -> punched in on face

The gap between levels must be visually obvious. The previous 85/65/45 split made normal and emphasis look too similar — viewers couldn't tell the difference. 100/70/45 creates three clearly distinct framings.

```python
# Portrait height fractions per label (how much source height to use)
portrait_height_frac = {
    "normal": 1.0,
    "emphasis": 0.70,
    "critical": 0.45,
}

frac = portrait_height_frac[section["label"]]

# Crop height is a fraction of source height; width enforces 9:16 ratio
ch = int(height * frac)
cw = int(ch * 9 / 16)
cw = cw - (cw % 2)  # must be even
ch = ch - (ch % 2)

# Horizontal: center on face
cx = max(0, min(face_cx - cw // 2, width - cw))

# Vertical: position face in the upper third of the crop.
# Offset the crop so face_cy lands at ~30% from the top of the crop region.
# This leaves head room above and torso/waist below.
face_target_y = int(ch * 0.30)
cy = max(0, min(face_cy - face_target_y, height - ch))
```

The output scale target for portrait is always `1080:1920`.

##### Build the filter_complex

**Single-angle mode** (no `source` field or all sections are `"primary"`):

```
[0:v]trim=start=S:end=E,setpts=PTS-STARTPTS,crop=CW:CH:CX:CY,scale=W:H:flags=lanczos[v0];
...
[v0][v1]...concat=n=N:v=1:a=0[outv]
```

Audio: map the full primary audio track directly with `-map 0:a` — do not use per-section `atrim`/`concat`. The sections are continuous and cover the full video, so the uncut audio track is already correct.

**Multi-angle mode** (sections have `source` field from `/swap-angles`):

For each section, check `section["source"]`:

- **Primary sections** (`"primary"`): Same as before — `[0:v]` with trim -> crop -> scale (zoom applied). The trim timestamps are the section's `start`/`end` from the sections JSON (trimmed primary timeline).
- **Secondary sections** (`"secondary_1"`, `"secondary_2"`): Use `[N:v]` where N is the ffmpeg input index. **No crop** — just trim and scale to output resolution. **The trim timestamps are computed by mapping the section's trimmed timestamps to the secondary file's timeline** using the helper functions above. Audio always comes from `[0:a]` (primary), using the original trimmed timestamps.

```python
# For a secondary section:
sec_idx = int(section["source"].split("_")[1]) - 1  # 0-based index
sec_start = trimmed_to_secondary(section["start"], segment_map, manifest, sec_idx)
sec_end = trimmed_to_secondary(section["end"], segment_map, manifest, sec_idx)
input_idx = sec_idx + 1  # ffmpeg input index (0 = primary, 1 = first secondary, ...)
```

```
# Primary section with zoom:
[0:v]trim=start=S:end=E,setpts=PTS-STARTPTS,crop=CW:CH:CX:CY,scale=W:H:flags=lanczos[v0];

# Secondary section — full frame, no zoom, DURATION-MATCHED to primary audio:
# Use trim=start=SEC_S:end=SEC_S+AUDIO_DUR (not mapped end time!)
[1:v]trim=start=SEC_S:end=SEC_S+DUR,setpts=PTS-STARTPTS,scale=W:H:flags=lanczos[v1];

...
# VIDEO only in concat — audio mapped separately:
[v0][v1]...concat=n=N:v=1:a=0[outv]
```

**Critical: use the full primary audio track, not per-section atrim/concat.**

The video concat builds the video-only stream. For audio, map the entire primary audio track directly with `-map 0:a` instead of cutting/rejoining audio segments. Per-section `atrim` + `concat` causes cumulative drift from frame-level duration mismatches — even sub-frame differences add up across 15+ sections and push audio out of sync.

```bash
ffmpeg -y -i <primary_trimmed> -i <secondary> \
  -filter_complex_script /tmp/zoom_filter_$$.txt \
  -map "[outv]" -map "0:a" \        # <-- full primary audio, no cutting
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 192k \
  <output>
```

**Secondary video duration matching**: For secondary sections, compute `sec_start` from the timestamp mapping but set `sec_end = sec_start + audio_duration` where `audio_duration = section["end"] - section["start"]` (the trimmed primary duration). Do NOT use the mapped end time — the segment map inflates the time range because it maps through removed silences, producing secondary clips that are longer than the corresponding audio. Duration-based trimming ensures the video concat total matches the primary audio track length.

Note: Secondary videos have no audio track — that's why audio always comes from the primary, regardless of which video provides the picture.

Scale targets:
- `-HD`: scale to 1920x1080
- `-4K`: scale back to source dimensions (zoom effect only)
- `-portrait`: scale to `1080:1920` (9:16 output)

Key details:
- Primary video: `trim` -> `setpts` -> `crop` -> `scale` (lanczos)
- Secondary video: `trim` (with mapped timestamps) -> `setpts` -> `scale` (lanczos) — no crop
- Audio: always from input 0 — `atrim` (with trimmed primary timestamps) -> `asetpts`
- Concat inputs must be interleaved: `[v0][a0][v1][a1]...`
- Write the filter to a temp file since it's too long for command line. Use a PID-unique filename (e.g., `/tmp/zoom_filter_{os.getpid()}.txt`) to avoid collisions with concurrent runs. Delete the temp file after ffmpeg completes.

#### Step 4: Render with ffmpeg

**Single-angle:**
```bash
ffmpeg -y -i <input> \
  -filter_complex_script /tmp/zoom_filter_$$.txt \
  -map "[outv]" -map "0:a" \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 192k \
  <output>
```

**Multi-angle** — pass all video files as inputs. The primary is input 0, and secondaries are inputs 1, 2, etc. Secondary file paths come from the sync manifest (the original, untrimmed secondary files):
```bash
ffmpeg -y -i <primary_trimmed> -i <secondary_1_original> [-i <secondary_2_original>] \
  -filter_complex_script /tmp/zoom_filter_$$.txt \
  -map "[outv]" -map "0:a" \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 192k \
  <output>
```

**Important**: Always use `-map "0:a"` to pass the full primary audio track through uncut. Never use per-section `atrim`/`concat` for audio — it causes cumulative sync drift.

Note: Use `-filter_complex_script <file>` to pass the filter from a file — this works across all ffmpeg versions. (ffmpeg 8.x+ also accepts `-/filter_complex <file>` as shorthand.)

#### Step 5: Report results

Print:
- Original video dimensions
- Output dimensions (source resolution for landscape, 1080x1920 for portrait)
- Face center position used
- Number of sections rendered and label distribution
- For multi-angle: number of secondary sections and the timestamp mapping used
- Output file path and duration

### Important notes

- **Landscape zoom**: a crop and scale operation — a 1.6x zoom crops to 1/1.6 of the frame centered on the face, then scales back up to the original resolution. Normal (1.0x) = full frame.
- **Portrait zoom**: crops a 9:16 slice from the 16:9 source, with the face positioned in the upper third. Normal = head-to-waist, emphasis = head-and-shoulders, critical = tight face. The output is always 1080x1920.
- All sections must be continuous (no gaps) and cover the full video duration.
- For landscape, the output maintains the original video resolution (e.g., 3840x2160). For portrait, output is always 1080x1920.
- Face detection uses 10 sample frames spread across the video. This assumes a relatively stationary speaker (talking head format). For videos with significant movement, more sophisticated per-section face tracking would be needed.
- Portrait mode assumes the source is landscape (16:9). If the source is already portrait or square, skip the portrait crop and just scale to 1080x1920.
- **Multi-angle sections get no zoom.** Secondary camera sections are rendered full-frame, scaled to the output resolution. The visual variety comes from the angle change itself, not from a zoom effect. Only primary camera sections (emphasis and critical) get the zoom crop.
- **Audio is always from the primary.** Even when the picture comes from a secondary camera, the audio track comes from input 0 (the primary with the quality mic). Secondary video files have no audio track.
- **Secondary timestamps are mapped, not direct.** In multi-angle mode, the section start/end times in the sections JSON are on the trimmed primary's timeline. To get the correct timestamps in a secondary's original file, use: `secondary_time = trimmed_to_original(trimmed_time) + manifest.overlap.primary_start - secondary.sync_offset`. The segment map handles the trimmed-to-original mapping (accounting for silence removal), and the manifest handles the original-to-secondary mapping (accounting for sync offsets).
- **Secondaries are never trimmed.** The produce-zoom step reads from the original secondary files and extracts the exact frames needed during its re-encoding pass. This avoids keyframe alignment issues from stream-copy trimming.
