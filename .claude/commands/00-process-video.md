---
name: process-video
description: Run the full 6-step video editing pipeline on a raw talking-head video to produce a polished final output
user_invocable: true
metadata:
  tags: video, pipeline, editing, automation, ffmpeg
---

## When to use

Use this skill when the user wants to run the full video editing pipeline on a raw video. Triggered by `/process-video <filename>` or requests like "process this video," "run the pipeline," or "edit this video."

## How to use

### Input

The user provides a filename as an argument, optionally followed by `-HD` or `-4K` to set output resolution. If no filename is provided, ask for one. The file should be in the current working directory or an absolute path.

- `-HD` (default): Final output scaled to 1920x1080
- `-4K`: Final output keeps source resolution (up to 3840x2160)
- If no flag is provided, defaults to `-HD`

Examples:
- `/process-video testvideo.mp4` → HD output (1920x1080)
- `/process-video testvideo.mp4 -HD` → HD output (1920x1080)
- `/process-video testvideo.mp4 -4K` → 4K output (source resolution)

### Prerequisites
- `ffmpeg` (standard) for steps 1-5
- `ffmpeg` at `/opt/homebrew/opt/ffmpeg/bin/ffmpeg` for step 6 (captions, via homebrew-ffmpeg tap with drawtext support)
- `whisper-cli` with model at `/opt/homebrew/share/whisper-cpp/models/ggml-medium.bin`
- `opencv-python-headless` (pip)
- Big Shoulders Display Bold 700 font at `/Users/chrislema/Library/Fonts/BigShouldersDisplay-Bold.ttf`

### Pipeline

Run these 6 steps in order. Each step takes the output of the previous step as input. All intermediate files go in the same directory as the input file.

Given input `<name>.<ext>`:

#### Step 1: Remove Silence (`/remove-silence`)
- **Input**: `<name>.<ext>`
- **Output**: `<name>_trimmed.<ext>`
- Detect silences > 0.5s using `silencedetect=noise=-30dB:d=0.5`
- Cut silences down to 0.3s natural pauses
- Use ffmpeg trim/atrim + concat filter

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
- Detect face position using OpenCV (10 sample frames, averaged)
- For each section, crop to zoom level centered on face, scale back to original resolution
- Render with libx264 CRF 18

#### Step 4: Correct Colors (`/correct-colors`)
- **Input**: `<name>_zoomed.<ext>`
- **Output**: `<name>_colorcorrected.<ext>`
- Apply: `colorbalance=rs=0.02:gs=-0.01:bs=-0.02,curves=m='0/0 0.25/0.20 0.75/0.82 1/1',eq=brightness=0.02:contrast=1.05:saturation=1.05`
- Video re-encoded, audio copied

#### Step 5: Master Audio (`/master-audio`)
- **Input**: `<name>_colorcorrected.<ext>`
- **Output**: `<name>_mastered.<ext>`
- Apply: `highpass=f=80,lowpass=f=14000,equalizer=f=3000:t=q:w=1.5:g=3,equalizer=f=200:t=q:w=1.0:g=2,acompressor=threshold=-21dB:ratio=3:attack=5:release=50,volume=0.6,loudnorm=I=-16:TP=-1.5:LRA=11`
- Video copied, audio re-encoded AAC 192k

#### Step 6: Add Captions (`/add-captions`)
- **Input**: `<name>_mastered.<ext>`
- **Output**: `<name>_final.mp4`
- **Resolution**: Pass the resolution flag (`-HD` or `-4K`) to this step. Default is `-HD` (1920x1080).
- Transcribe with whisper-cli
- Break into max 6 words per caption, ALL CAPS
- Big Shoulders Display Bold 700, white text on black box (70% opacity)
- Font size calculated from **target** dimensions (not source): `target_width * 0.0495`, centered at `target_height * 0.80`
- If downscaling, prepend `scale=1920:1080` to the filter chain before drawtext filters
- **Must use homebrew-ffmpeg tap** (`/opt/homebrew/opt/ffmpeg/bin/ffmpeg`) for drawtext filter
- Output as `_final.mp4` (always mp4 regardless of input format)

### Output

The final file is `<name>_final.mp4` in the same directory as the input.

After completion, report:
- Original duration vs final duration
- Time saved from silence removal
- Number of zoom sections and label distribution
- Confirm all 6 steps completed

#### Step 7: Clean Artifacts (`/clean-artifacts`)
- Delete intermediate files: `_trimmed`, `_trimmed_sections.json`, `_zoomed`, `_colorcorrected`, `_mastered`, `_captioned`
- Keep only the original `<name>.<ext>` and `<name>_final.mp4`
- Report number of files deleted and disk space recovered
- No confirmation needed when called from this pipeline

### Error handling

If any step fails, stop the pipeline and report which step failed and why. Do not continue to subsequent steps. Do not run cleanup if the pipeline failed.
