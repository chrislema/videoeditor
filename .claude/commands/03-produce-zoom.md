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
- **input**: Path to the video file (if not provided, infer from JSON metadata or ask)
- **sections_json**: Path to the sections JSON file (if not provided, look for `<input_basename>_sections.json`)
- **output**: Path for the output file (default: `<input_basename>_zoomed.mp4`)

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

#### Step 2: Detect face position

Use OpenCV's Haar cascade face detector to find the average face center across sampled frames:

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

For each section, calculate the crop region based on the zoom level:

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

Build the filter_complex with trim/crop/scale per section:

```
[0:v]trim=start=S:end=E,setpts=PTS-STARTPTS,crop=CW:CH:CX:CY,scale=W:H:flags=lanczos[v0];
[0:a]atrim=start=S:end=E,asetpts=PTS-STARTPTS[a0];
...
[v0][a0][v1][a1]...concat=n=N:v=1:a=1[outv][outa]
```

Key details:
- Video: `trim` -> `setpts` -> `crop` -> `scale` (back to original dimensions using lanczos)
- Audio: `atrim` -> `asetpts`
- Concat inputs must be interleaved: `[v0][a0][v1][a1]...`
- Write the filter to a temp file since it's too long for command line

#### Step 4: Render with ffmpeg

```bash
ffmpeg -y -i <input> \
  -/filter_complex /tmp/zoom_filter.txt \
  -map "[outv]" -map "[outa]" \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 192k \
  <output>
```

Note: Use `-/filter_complex <file>` for ffmpeg 8.x+. For older ffmpeg, use `-filter_complex_script <file>`.

#### Step 5: Report results

Print:
- Original video dimensions
- Face center position used
- Number of sections rendered and label distribution
- Output file path and duration

### Important notes

- The zoom is a **crop and scale** operation: a 1.6x zoom crops to 1/1.6 of the frame centered on the face, then scales back up to the original resolution. This simulates "zooming in."
- Normal (1.0x) means no crop — full frame is used.
- All sections must be continuous (no gaps) and cover the full video duration.
- The output maintains the original video resolution (e.g., 3840x2160).
- Face detection uses 10 sample frames spread across the video. This assumes a relatively stationary speaker (talking head format). For videos with significant movement, more sophisticated per-section face tracking would be needed.
