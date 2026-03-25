---
name: process-video
description: Run the full 7-step video editing pipeline on a raw talking-head video to produce a polished final output
user_invocable: true
metadata:
  tags: video, pipeline, editing, automation, ffmpeg
---

## When to use

Use this skill when the user wants to run the full video editing pipeline on a raw video. Triggered by `/process-video <filename>` or requests like "process this video," "run the pipeline," or "edit this video."

## How to use

### Input

The user provides a filename as an argument, optionally followed by one or more flags. If no filename is provided, ask for one. The file should be in the current working directory or an absolute path. Flags are case-insensitive and can be combined in any order.

**Resolution flags** (mutually exclusive — last one wins if multiple are given):
- `-HD` (default): Final output scaled to 1920x1080 (landscape 16:9)
- `-4K`: Final output keeps source resolution (up to 3840x2160, landscape)
- `-portrait`: Final output scaled to 1080x1920 (vertical 9:16, for Reels/Shorts/LinkedIn)
- If no resolution flag is provided, defaults to `-HD`

**Option flags** (combinable with any resolution flag):
- `-nocaptions`: Skip the captions step (step 6). The mastered video becomes the final output directly. Useful for videos where captions aren't needed or will be added separately.

**Flag parsing**: Normalize all flags to lowercase before matching. `-HD`, `-hd`, `-Hd` are all equivalent. `-NoCaptions`, `-nocaptions`, `-NOCAPTIONS` are all equivalent. Flags can appear in any order after the filename.

Examples:
- `/process-video testvideo.mp4` → HD output with captions
- `/process-video testvideo.mp4 -HD` → HD output with captions
- `/process-video testvideo.mp4 -4K` → 4K output with captions
- `/process-video testvideo.mp4 -portrait` → Portrait output (1080x1920) with captions
- `/process-video testvideo.mp4 -nocaptions` → HD output, no captions
- `/process-video testvideo.mp4 -portrait -nocaptions` → Portrait output, no captions
- `/process-video testvideo.mp4 -hd -nocaptions` → HD output, no captions
- `/process-video testvideo.mp4 -4K -nocaptions` → 4K output, no captions

### Prerequisites
- `ffmpeg` (standard) for steps 1-5
- `ffmpeg-full` at `/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg` for step 6 (captions, with drawtext/libfreetype support) — not needed if `-nocaptions` is used
- `whisper-cli` with model at `/opt/homebrew/share/whisper-cpp/models/ggml-medium.bin`
- `opencv-python-headless` (pip)
- Big Shoulders Display Bold 700 font at `~/Library/Fonts/BigShouldersDisplay-Bold.ttf` (resolve `~` via `os.path.expanduser()` at runtime) — not needed if `-nocaptions` is used

### Pre-flight check

Before running any pipeline step, validate that all required tools and files are present. **Stop immediately and report what's missing** if any check fails.

```python
import subprocess, os, sys

errors = []

# 1. ffmpeg (standard)
if subprocess.run(["which", "ffmpeg"], capture_output=True).returncode != 0:
    errors.append("ffmpeg not found. Install with: brew install ffmpeg")

# 2. whisper-cli (try common binary names)
whisper_bin = None
for name in ["whisper-cli", "whisper-cpp", "main"]:
    if subprocess.run(["which", name], capture_output=True).returncode == 0:
        whisper_bin = name
        break
if not whisper_bin:
    errors.append("whisper-cli not found. Install with: brew install whisper-cpp")

# 3. Whisper model
model_path = "/opt/homebrew/share/whisper-cpp/models/ggml-medium.bin"
if not os.path.exists(model_path):
    errors.append(f"Whisper model not found at {model_path}. Run: cd /opt/homebrew/share/whisper-cpp/models && bash download-ggml-model.sh medium")

# 4. OpenCV
try:
    subprocess.run([sys.executable, "-c", "import cv2"], capture_output=True, check=True)
except subprocess.CalledProcessError:
    errors.append("opencv-python-headless not found. Install with: pip3 install opencv-python-headless")

# 5. Captions prerequisites (only if captions are enabled)
if not nocaptions:
    # ffmpeg with drawtext support
    ffmpeg_full = "/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg"
    if not os.path.exists(ffmpeg_full):
        errors.append(f"ffmpeg-full not found at {ffmpeg_full}. Install with: brew install homebrew-ffmpeg/ffmpeg/ffmpeg-full")

    # Font file
    font_path = os.path.expanduser("~/Library/Fonts/BigShouldersDisplay-Bold.ttf")
    if not os.path.exists(font_path):
        errors.append(f"Caption font not found at {font_path}. Download from Google Fonts: https://fonts.google.com/specimen/Big+Shoulders+Display")

if errors:
    print("Pre-flight check failed:")
    for e in errors:
        print(f"  - {e}")
    # STOP — do not proceed with the pipeline
```

### Pipeline

Run these steps in order. Each step takes the output of the previous step as input. All intermediate files go in the same directory as the input file.

Given input `<name>.<ext>`:

#### Step 1: Remove Silence (`/remove-silence`)
- **Input**: `<name>.<ext>`
- **Output**: `<name>_trimmed.<ext>`
- Detect silences > 0.5s using `silencedetect=noise=-30dB:d=0.5`
- Cut silences down to 0.3s natural pauses
- Extract segments with `-c copy` (no re-encoding) and concatenate via concat demuxer

#### Step 2: Label Sections (`/label-sections`)
- **Input**: `<name>_trimmed.<ext>`
- **Output**: `<name>_trimmed_sections.json`
- Transcribe with whisper-cli
- Break into sections: min 3s, max 6s per section
- Label each section as normal (1.0x), emphasis (1.25x), or critical (1.6x) based on content analysis
- ~40% normal, ~35% emphasis, ~25% critical

#### Step 3: Produce Zoom (`/produce-zoom`)
- **Input**: `<name>_trimmed.<ext>` + `<name>_trimmed_sections.json`
- **Output**: `<name>_zoomed.<ext>`
- **Resolution**: Pass the resolution flag (`-HD`, `-4K`, or `-portrait`) to this step.
- Detect face position using OpenCV (10 sample frames, averaged)
- For landscape: crop to zoom level centered on face, scale back to original resolution
- For `-portrait`: crop a 9:16 region from the 16:9 source centered on face. Normal = head-to-waist (~85% height), emphasis = head-and-shoulders (~65% height), critical = tight face (~45% height). Face positioned in upper third with torso below.
- Render with libx264 CRF 18

#### Step 4: Correct Colors (`/correct-colors`)
- **Input**: `<name>_zoomed.<ext>`
- **Output**: `<name>_colorcorrected.<ext>`
- **Resolution**: Pass the resolution flag to this step. For `-portrait`, output is already 1080x1920 from step 3.
- Apply: `colorbalance=rs=0.02:gs=-0.01:bs=-0.02,curves=m='0/0 0.25/0.20 0.75/0.82 1/1',eq=brightness=0.02:contrast=1.05:saturation=1.05`
- Video re-encoded, audio copied

#### Step 5: Master Audio (`/master-audio`)
- **Input**: `<name>_colorcorrected.<ext>`
- **Output**: `<name>_mastered.<ext>`
- Apply: `highpass=f=80,lowpass=f=14000,equalizer=f=3000:t=q:w=1.5:g=3,equalizer=f=200:t=q:w=1.0:g=2,acompressor=threshold=-21dB:ratio=3:attack=5:release=50,volume=0.6,loudnorm=I=-16:TP=-1.5:LRA=11`
- Video copied, audio re-encoded AAC 192k

#### Step 6: Add Captions (`/add-captions`)

**Skip this step if `-nocaptions` flag was provided.** When skipping, copy or rename `<name>_mastered.<ext>` to `<name>_final.mp4` instead:
```bash
ffmpeg -y -i <name>_mastered.<ext> -c copy <name>_final.mp4
```

If captions are enabled (the default):
- **Input**: `<name>_mastered.<ext>`
- **Output**: `<name>_final.mp4`
- **Resolution**: Pass the resolution flag (`-HD`, `-4K`, or `-portrait`) to this step. Default is `-HD` (1920x1080).
- Transcribe with whisper-cli
- For landscape: break into max 6 words per caption, ALL CAPS
- For `-portrait`: break into max 3 words per caption (narrower frame), ALL CAPS
- Big Shoulders Display Bold 700, white text on black box (70% opacity)
- Font size calculated from **target** dimensions (not source): `target_width * 0.0495`, centered at `target_height * 0.80`
- For `-portrait`: target is 1080x1920, font size from `target_width * 0.065` (larger relative to narrow frame), centered at `target_height * 0.75` (higher to avoid thumb zone)
- If downscaling, prepend appropriate `scale=` filter to the filter chain before drawtext filters
- **Must use ffmpeg-full** (`/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg`) for drawtext filter
- Output as `_final.mp4` (always mp4 regardless of input format)

### Output

The final file is `<name>_final.mp4` in the same directory as the input.

After completion, report:
- Original duration vs final duration
- Time saved from silence removal
- Number of zoom sections and label distribution
- Whether captions were included or skipped
- Confirm all steps completed

#### Step 7: Clean Artifacts (`/clean-artifacts`)
- Delete intermediate files: `_trimmed`, `_trimmed_sections.json`, `_zoomed`, `_colorcorrected`, `_mastered`, `_captioned`
- Keep only the original `<name>.<ext>` and `<name>_final.mp4`
- Report number of files deleted and disk space recovered
- No confirmation needed when called from this pipeline

### Error handling

If any step fails, stop the pipeline and report which step failed and why. Do not continue to subsequent steps. Do not run cleanup if the pipeline failed.
