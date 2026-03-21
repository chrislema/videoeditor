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

The user provides a filename as an argument. If no filename is provided, ask for one. The file should be in the current working directory or an absolute path.

Example: `/process-video testvideo.mp4`

### Prerequisites
- `ffmpeg` (standard) for steps 1-5
- `ffmpeg-full` at `/opt/homebrew/opt/ffmpeg-full/bin/ffmpeg` for step 6 (captions)
- `whisper-cli` with model at `/opt/homebrew/share/whisper-cpp/models/ggml-medium.bin`
- `opencv-python-headless` (pip)
- Big Shoulders Display Bold 700 font at `/Users/chrislema/Library/Fonts/BigShouldersDisplay-700.ttf`

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
- Transcribe with whisper-cli
- Break into max 6 words per caption, ALL CAPS
- Big Shoulders Display Bold 700, white text on black box (70% opacity)
- Font size: `width * 0.062`, centered at `height * 0.80`
- **Must use ffmpeg-full** for drawtext filter
- Output as `_final.mp4` (always mp4 regardless of input format)

### Output

The final file is `<name>_final.mp4` in the same directory as the input.

After completion, report:
- Original duration vs final duration
- Time saved from silence removal
- Number of zoom sections and label distribution
- Confirm all 6 steps completed

### Cleanup

Do NOT delete intermediate files automatically. The user may want to review or re-run individual steps. Only delete intermediates if the user explicitly asks.

### Error handling

If any step fails, stop the pipeline and report which step failed and why. Do not continue to subsequent steps.
