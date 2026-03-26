---
name: master-audio
description: Apply broadcast-ready audio mastering chain to video files using ffmpeg for clear, warm talking-head sound
user_invocable: true
metadata:
  tags: video, audio, mastering, eq, compression, ffmpeg
---

## When to use

Use this skill when the user wants to master, clean up, or improve the audio on a video. Triggered by requests like "master the audio," "fix the audio," "clean up audio," "audio sounds bad," or "master-audio."

## How to use

### Prerequisites
- `ffmpeg` must be installed

### Parameters
The user may optionally specify:
- **input**: Path to the video file (if not provided, ask)
- **output**: Path for the output file (default: strip `_colorcorrected` from the input basename, then append `_mastered.<ext>` — e.g., `mainvideo_colorcorrected.mp4` → `mainvideo_mastered.mp4`)

### Default mastering chain

The full ffmpeg audio filter chain, optimized for talking-head videos:

```
highpass=f=80,lowpass=f=14000,equalizer=f=3000:t=q:w=1.5:g=3,equalizer=f=200:t=q:w=1.0:g=2,acompressor=threshold=-21dB:ratio=3:attack=5:release=50,volume=0.6,loudnorm=I=-16:TP=-1.5:LRA=11
```

What each stage does:

1. **Highpass 80Hz** — removes room rumble, AC hum, and low-frequency noise
2. **Lowpass 14kHz** — removes high-frequency hiss and sibilance artifacts
3. **Presence EQ: +3dB at 3kHz** — boosts voice clarity and intelligibility (Q=1.5)
4. **Warmth EQ: +2dB at 200Hz** — adds fullness to the voice (Q=1.0)
5. **Compressor: 3:1 ratio, -21dB threshold** — evens out volume dynamics (attack=5ms, release=50ms)
6. **Volume 0.6** — gain reduction to prevent clipping after EQ boosts, gives headroom before loudnorm
7. **Loudness normalization: -16 LUFS, true peak -1.5dB, LRA=11** — broadcast-ready levels

### Process

#### Step 1: Apply the mastering chain

```bash
ffmpeg -y -i <input> \
  -af "highpass=f=80,lowpass=f=14000,equalizer=f=3000:t=q:w=1.5:g=3,equalizer=f=200:t=q:w=1.0:g=2,acompressor=threshold=-21dB:ratio=3:attack=5:release=50,volume=0.6,loudnorm=I=-16:TP=-1.5:LRA=11" \
  -c:v copy \
  -c:a aac -b:a 192k \
  <output>
```

Key details:
- Video is copied (`-c:v copy`) since audio mastering doesn't affect video
- Audio is re-encoded with AAC at 192kbps
- All filters run as a single chain in order

#### Step 2: Report results

Print the input file, output file, and confirm the mastering chain applied.

### Adjustments

If the user wants to tweak the sound:

**Voice too thin/tinny:**
- Increase warmth: raise `g` on the 200Hz EQ (e.g., `g=3` or `g=4`)
- Lower the highpass frequency (e.g., `f=60`)

**Voice too boomy/muddy:**
- Decrease warmth: lower `g` on the 200Hz EQ (e.g., `g=1` or `g=0`)
- Raise the highpass frequency (e.g., `f=100`)

**Voice not clear enough:**
- Increase presence: raise `g` on the 3kHz EQ (e.g., `g=4` or `g=5`)

**Too much compression (sounds squashed):**
- Lower the ratio (e.g., `ratio=2`)
- Lower the threshold (e.g., `threshold=-24dB`)

**Not enough compression (volume jumps around):**
- Raise the ratio (e.g., `ratio=4`)
- Raise the threshold (e.g., `threshold=-18dB`)

**Target loudness:**
- YouTube/podcasts: -16 LUFS (default)
- Spotify/streaming: -14 LUFS
- Broadcast TV: -24 LUFS

### Important notes
- This skill should typically be the **last step** in the pipeline, after silence removal, zoom, and color correction
- The chain is designed for solo voice recordings. For multi-speaker or music content, different settings may be needed.
- The `volume=0.6` stage is critical — it prevents the EQ boosts from clipping before loudnorm normalizes the final level
