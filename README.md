# Video Editor — Claude Code Skills Pipeline

An 8-step automated video editing pipeline built as Claude Code skills. Designed for talking-head videos — takes a raw recording and produces a polished final output with silence removed, dynamic zoom, color correction, audio mastering, and burned-in captions.

## What it does

Given a raw video file, `/process-video <filename> [-HD|-4K|-portrait] [-nocaptions]` runs the full pipeline. Output defaults to **1080p HD** unless another flag is specified. Flags can be combined in any order.

1. **Remove Silence** — Detects silent gaps longer than 0.5s using ffmpeg's `silencedetect` and trims them down to 0.3s natural pauses. Extracts segments with stream copy (no re-encoding) and concatenates via the concat demuxer.

2. **Label Sections** — Transcribes the trimmed video with whisper-cli, then segments the transcript into 3–6 second chunks. Each chunk is labeled based on rhetorical analysis:
   - **normal** (1.0x) — setup, context, transitions (~40%)
   - **emphasis** (1.25x) — key points, rhetorical questions, rising energy (~35%)
   - **critical** (1.6x) — thesis statements, punchlines, emotional peaks (~25%)

3. **Produce Zoom** — Uses OpenCV's Haar cascade face detector across 10 sampled frames to find the average face position. For landscape: crops the frame to the corresponding zoom level centered on the face, then scales back to original resolution (normal = full frame, critical = tight face crop). For `-portrait`: crops a 9:16 region from the 16:9 source with the face in the upper third — normal = head-to-waist (~85% height), emphasis = head-and-shoulders (~65%), critical = tight face (~45%). Output is 1080x1920.

4. **Correct Colors** — Applies a "warm-punch" color grade tuned for indoor talking-head footage: subtle warm color balance shift, mild S-curve contrast, slight brightness and saturation lift.

5. **Master Audio** — Full broadcast-ready audio chain: highpass/lowpass filtering, presence EQ boost at 3kHz, warmth at 200Hz, 3:1 compression, gain staging, and loudness normalization to -16 LUFS.

6. **Add Captions** — Transcribes again with whisper-cli and burns in captions using ffmpeg's drawtext filter. White text on a 70% opacity black box, Big Shoulders Display Bold 700. For landscape: 6-word ALL CAPS chunks at 80% of frame height. For `-portrait`: 3-word chunks with larger relative font, positioned at 75% height to avoid the mobile thumb zone.

7. **Review Final Output** — Opens the final video for review. If you're happy, proceed to cleanup. If not, all intermediate files are preserved so you can re-run individual steps.

8. **Clean Artifacts** — Deletes all intermediate files (`_trimmed`, `_zoomed`, `_colorcorrected`, `_mastered`, `_sections.json`), keeping only the original and `_final.mp4`.

## Prerequisites

### Homebrew packages

```bash
# Standard ffmpeg (used for steps 1–5)
brew install ffmpeg

# ffmpeg-full with drawtext/libfreetype support (required for step 6: captions)
brew install homebrew-ffmpeg/ffmpeg/ffmpeg-full

# whisper.cpp CLI for transcription (steps 2 and 6)
brew install whisper-cpp
```

After installing whisper-cpp, verify the medium model is available:

```bash
ls /opt/homebrew/share/whisper-cpp/models/ggml-medium.bin
```

If the model isn't there, download it:

```bash
# whisper-cpp includes a download script
cd /opt/homebrew/share/whisper-cpp/models
bash download-ggml-model.sh medium
```

### Python packages

```bash
pip3 install opencv-python-headless
```

OpenCV is used for face detection in step 3 (produce-zoom). The `headless` variant is sufficient — no GUI needed.

### Font

The caption step uses **Big Shoulders Display Bold 700** from Google Fonts. Download the static TTF and place it in your user fonts directory:

```bash
# Download from Google Fonts
curl -L -o /tmp/BigShouldersDisplay.zip \
  "https://fonts.google.com/download?family=Big+Shoulders+Display"

# Extract and install the 700 weight
unzip /tmp/BigShouldersDisplay.zip -d /tmp/BigShouldersDisplay
cp /tmp/BigShouldersDisplay/static/BigShouldersDisplay-Bold.ttf \
  ~/Library/Fonts/BigShouldersDisplay-Bold.ttf
```

The pipeline expects the font at:
```
~/Library/Fonts/BigShouldersDisplay-Bold.ttf
```

**Important**: `ffmpeg-full` uses fontconfig (`--enable-libfontconfig`) to resolve fonts by name. The drawtext filter must use `font='Big Shoulders Display'`, NOT `fontfile='/path/to/file.ttf'`. The `fontfile` parameter is silently ignored when fontconfig is enabled, causing a fallback to Verdana (which is wider and breaks the caption width constraints). After installing the font, run `fc-cache -fv` to update the fontconfig cache.

### Verify everything

Quick check that all prerequisites are in place:

```bash
# ffmpeg (standard)
which ffmpeg && ffmpeg -version | head -1

# ffmpeg-full with drawtext
ls /opt/homebrew/opt/ffmpeg-full/bin/ffmpeg

# whisper-cli
which whisper-cli

# whisper model
ls /opt/homebrew/share/whisper-cpp/models/ggml-medium.bin

# OpenCV
python3 -c "import cv2; print(f'OpenCV {cv2.__version__}')"

# Font
ls ~/Library/Fonts/BigShouldersDisplay-Bold.ttf
```

## Usage

These are Claude Code skills. Place the `.md` files in your Claude Code skills directory and invoke:

```
/process-video myrecording.mp4              # → 1080p HD output (default)
/process-video myrecording.mp4 -HD          # → 1080p HD output
/process-video myrecording.mp4 -4K          # → 4K output (source resolution)
/process-video myrecording.mp4 -portrait    # → Portrait 1080x1920 (9:16 vertical)
/process-video myrecording.mp4 -nocaptions  # → HD output, no captions
/process-video myrecording.mp4 -portrait -nocaptions  # → Portrait, no captions
```

Or run individual steps:

```
/remove-silence myrecording.mp4
/label-sections myrecording_trimmed.mp4
/produce-zoom myrecording_trimmed.mp4
/correct-colors myrecording_zoomed.mp4
/master-audio myrecording_colorcorrected.mp4
/add-captions myrecording_mastered.mp4
/clean-artifacts myrecording
```

### File naming convention

Each step appends a suffix and feeds into the next:

```
myrecording.mp4
  → myrecording_trimmed.mp4
  → myrecording_trimmed_sections.json
  → myrecording_zoomed.mp4
  → myrecording_colorcorrected.mp4
  → myrecording_mastered.mp4
  → myrecording_final.mp4
```

After cleanup, only the original and `_final.mp4` remain.

## Platform notes

- Built and tested on **macOS** (Apple Silicon / Homebrew)
- ffmpeg paths assume Homebrew on ARM Macs (`/opt/homebrew/...`). Intel Macs use `/usr/local/...` — update paths accordingly
- The font path is macOS-specific (`~/Library/Fonts/`). On Linux, adjust to `~/.local/share/fonts/` or `/usr/share/fonts/`
- `whisper-cli` binary name may vary by installation method — some installs use `whisper-cpp` or `main` instead
