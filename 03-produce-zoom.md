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
- **output**: Path for the output file (default: `<input_basename>_zoomed.mp4`)
- **resolution**: `-HD` (default), `-4K`, or `-portrait` (1080x1920 vertical 9:16)

### Process

#### Step 1: Load sections JSON and video metadata

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
- `"primary"` → input file (ffmpeg input index 0)
- `"secondary_1"` → first secondary file (ffmpeg input index 1)
- `"secondary_2"` → second secondary file (ffmpeg input index 2)

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

- **normal** (head-to-waist): use ~85% of source height → generous torso framing
- **emphasis** (head-and-shoulders): use ~65% of source height → tighter
- **critical** (tight face): use ~45% of source height → punched in on face

```python
# Portrait height fractions per label (how much source height to use)
portrait_height_frac = {
    "normal": 0.85,
    "emphasis": 0.65,
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
[0:a]atrim=start=S:end=E,asetpts=PTS-STARTPTS[a0];
...
[v0][a0][v1][a1]...concat=n=N:v=1:a=1[outv][outa]
```

**Multi-angle mode** (sections have `source` field from `/swap-angles`):

For each section, check `section["source"]`:

- **Primary sections** (`"primary"`): Same as before — `[0:v]` with trim → crop → scale (zoom applied).
- **Secondary sections** (`"secondary_1"`, `"secondary_2"`): Use `[N:v]` where N is the ffmpeg input index. **No crop** — just trim and scale to output resolution. Audio always comes from `[0:a]` (primary).

```
# Primary section with zoom:
[0:v]trim=start=S:end=E,setpts=PTS-STARTPTS,crop=CW:CH:CX:CY,scale=W:H:flags=lanczos[v0];
[0:a]atrim=start=S:end=E,asetpts=PTS-STARTPTS[a0];

# Secondary section — full frame, no zoom:
[1:v]trim=start=S:end=E,setpts=PTS-STARTPTS,scale=W:H:flags=lanczos[v1];
[0:a]atrim=start=S:end=E,asetpts=PTS-STARTPTS[a1];

...
[v0][a0][v1][a1]...concat=n=N:v=1:a=1[outv][outa]
```

Note: Secondary videos have no audio track — that's why audio is always `[0:a]` (primary) for every section, regardless of which video provides the picture.

Scale targets:
- `-HD`: scale to 1920x1080
- `-4K`: scale back to source dimensions (zoom effect only)
- `-portrait`: scale to `1080:1920` (9:16 output)

Key details:
- Primary video: `trim` -> `setpts` -> `crop` -> `scale` (lanczos)
- Secondary video: `trim` -> `setpts` -> `scale` (lanczos) — no crop
- Audio: always from input 0 — `atrim` -> `asetpts`
- Concat inputs must be interleaved: `[v0][a0][v1][a1]...`
- Write the filter to a temp file since it's too long for command line. Use a PID-unique filename (e.g., `/tmp/zoom_filter_{os.getpid()}.txt`) to avoid collisions with concurrent runs. Delete the temp file after ffmpeg completes.

#### Step 4: Render with ffmpeg

**Single-angle:**
```bash
ffmpeg -y -i <input> \
  -filter_complex_script /tmp/zoom_filter_$$.txt \
  -map "[outv]" -map "[outa]" \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 192k \
  <output>
```

**Multi-angle** — pass all video files as inputs:
```bash
ffmpeg -y -i <primary> -i <secondary_1> [-i <secondary_2>] \
  -filter_complex_script /tmp/zoom_filter_$$.txt \
  -map "[outv]" -map "[outa]" \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 192k \
  <output>
```

Note: Use `-filter_complex_script <file>` to pass the filter from a file — this works across all ffmpeg versions. (ffmpeg 8.x+ also accepts `-/filter_complex <file>` as shorthand.)

#### Step 5: Report results

Print:
- Original video dimensions
- Output dimensions (source resolution for landscape, 1080x1920 for portrait)
- Face center position used
- Number of sections rendered and label distribution
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
