---
name: correct-colors
description: Apply color correction to video files using ffmpeg filters for warm, punchy talking-head look
user_invocable: true
metadata:
  tags: video, color, correction, grading, ffmpeg
---

## When to use

Use this skill when the user wants to color correct a video. Triggered by requests like "color correct," "color grade," "fix the colors," "make it warmer," or "color-correct."

## How to use

### Prerequisites
- `ffmpeg` must be installed

### Parameters
The user may optionally specify:
- **input**: Path to the video file (if not provided, ask)
- **output**: Path for the output file (default: strip `_zoomed` from the input basename, then append `_colorcorrected.<ext>` — e.g., `mainvideo_zoomed.mp4` → `mainvideo_colorcorrected.mp4`)
- **preset**: Which color preset to use (default: "warm-punch")

### Default preset: warm-punch

These are the default ffmpeg video filters, tuned for talking-head videos with standard indoor lighting:

```
colorbalance=rs=0.02:gs=-0.01:bs=-0.02
curves=m='0/0 0.25/0.20 0.75/0.82 1/1'
eq=brightness=0.02:contrast=1.05:saturation=1.05
```

What each filter does:
- **colorbalance** — subtle warm shift: +0.02 red shadows, -0.01 green, -0.02 blue. Warms skin tones without going orange.
- **curves** — master curve that slightly crushes shadows and lifts highlights for a mild S-curve contrast.
- **eq** — small brightness bump (+0.02), gentle contrast boost (1.05x), and slight saturation lift (1.05x).

### Process

#### Step 1: Apply filters

Chain the three filters as a single `-vf` argument, comma-separated:

```bash
ffmpeg -y -i <input> \
  -vf "colorbalance=rs=0.02:gs=-0.01:bs=-0.02,curves=m='0/0 0.25/0.20 0.75/0.82 1/1',eq=brightness=0.02:contrast=1.05:saturation=1.05" \
  -c:v libx264 -preset fast -crf 18 \
  -c:a copy \
  <output>
```

Key details:
- Audio is copied (`-c:a copy`) since color correction doesn't affect audio
- Video is re-encoded with libx264 at CRF 18 for high quality
- Filters are applied in order: color balance first, then curves, then EQ

#### Step 2: Report results

Print the input file, output file, and confirm the filters applied.

### Adjustments

If the user wants to tweak the look, adjust these values:

**Warmer/Cooler:**
- Warmer: increase `rs` (red shadows), decrease `bs` (blue shadows)
- Cooler: decrease `rs`, increase `bs`

**More/Less contrast:**
- More: widen the curves gap (e.g., `0.25/0.15 0.75/0.85`) and increase `contrast` in eq
- Less: flatten the curves (e.g., `0.25/0.23 0.75/0.78`) and decrease `contrast`

**Brightness:**
- Adjust `brightness` in eq (range: -1.0 to 1.0, 0 is neutral)

**Saturation:**
- Adjust `saturation` in eq (1.0 is neutral, higher is more vivid, lower is more muted)

### Important notes
- This skill should typically be the **last step** in the pipeline, after silence removal and zoom processing
- The filters are resolution-agnostic — they work identically on landscape (16:9), portrait (9:16), and any other aspect ratio. No special handling is needed for `-portrait` mode; the color correction filters apply the same way regardless of frame dimensions.
- The filters are designed for talking-head content. Different video types (outdoor, product shots, screencasts) may need different settings.
- If the input video already has a color grade or LUT applied, these filters stack on top — results may be too warm or contrasty. Ask the user about their source footage.
