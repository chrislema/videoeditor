---
name: process-video
description: Run the full video editing pipeline on a raw talking-head video to produce a polished final output, with optional multi-angle support
user_invocable: true
metadata:
  tags: video, pipeline, editing, automation, ffmpeg, multi-angle
---

## When to use

Use this skill when the user wants to run the full video editing pipeline on a raw video. Supports single-camera and multi-angle (2-3 cameras) workflows. Triggered by `/process-video <filename>` or requests like "process this video," "run the pipeline," or "edit this video."

## How to use

### Input

The user provides a filename as an argument, optionally followed by additional video files (secondary camera angles) and flags. If no filename is provided, ask for one. The file should be in the current working directory or an absolute path. Additional video file arguments after the primary are treated as secondary angles for multi-angle editing.

Flags are case-insensitive and can be combined in any order.

**Resolution flags** (mutually exclusive — last one wins if multiple are given):
- `-HD` (default): Final output scaled to 1920x1080 (landscape 16:9)
- `-4K`: Final output keeps source resolution (up to 3840x2160, landscape)
- `-portrait`: Final output scaled to 1080x1920 (vertical 9:16, for Reels/Shorts/LinkedIn)
- If no resolution flag is provided, defaults to `-HD`

**Option flags** (combinable with any resolution flag):
- `-nocaptions`: Skip the captions step (step 6). The mastered video becomes the final output directly. Useful for videos where captions aren't needed or will be added separately.

**Flag parsing**: Normalize all flags to lowercase before matching. `-HD`, `-hd`, `-Hd` are all equivalent. `-NoCaptions`, `-nocaptions`, `-NOCAPTIONS` are all equivalent. Flags can appear in any order after the filename. Arguments that are not flags and end in a video extension (`.mp4`, `.mov`, `.mkv`, `.webm`, `.mpg`, `.avi`, `.m4v`) are treated as secondary video files.

**Naming convention**: All intermediate and output files use the **original primary's base name** (filename without extension) as the prefix. For example, given `mainvideo.mp4`, every file in the pipeline starts with `mainvideo_`: `mainvideo_synced.mp4`, `mainvideo_trimmed.mp4`, `mainvideo_zoomed.mp4`, etc. Each step replaces (not appends to) the previous step's suffix. The `_synced` suffix from sync-feeds is stripped by remove-silence so it doesn't cascade into downstream names.

Examples:
- `/process-video testvideo.mp4` -> HD output with captions (single camera)
- `/process-video testvideo.mp4 -HD` -> HD output with captions
- `/process-video testvideo.mp4 -4K` -> 4K output with captions
- `/process-video testvideo.mp4 -portrait` -> Portrait output (1080x1920) with captions
- `/process-video testvideo.mp4 -nocaptions` -> HD output, no captions
- `/process-video testvideo.mp4 -portrait -nocaptions` -> Portrait output, no captions
- `/process-video testvideo.mp4 side_angle.mp4` -> Multi-angle HD output with captions
- `/process-video testvideo.mp4 side_angle.mp4 -4K` -> Multi-angle 4K output
- `/process-video testvideo.mp4 angle2.mp4 angle3.mp4 -HD` -> 3-camera multi-angle HD output

### Prerequisites
- `ffmpeg` and `ffprobe` on PATH for steps 1-5
- An `ffmpeg` with the `drawtext` filter for captions. Homebrew `ffmpeg` 9+ and most Linux builds include it; older Homebrew `ffmpeg` does not, so fall back to `ffmpeg-full` (`brew install ffmpeg-full`)
- `whisper-cli` (Homebrew formula `whisper-cpp`) and the `ggml-medium.bin` model
- `opencv-python-headless<5` (pip) — OpenCV 5 dropped the Haar cascade face detector used in step 3
- A static Big Shoulders Display Bold (700) font, referenced with `fontfile=` — not needed if `-nocaptions` is used
- `scipy` and `numpy` (pip) — only required when secondary angles are provided (for audio cross-correlation sync)

### Locating tools

Never hard-code install paths — they differ between Apple Silicon, Intel Macs, and Linux. Resolve them with these functions, which check an environment variable override first, then the common install locations. Use the resolved values everywhere a step below says `<FFMPEG_DRAWTEXT>`, `<WHISPER_BIN>`, `<WHISPER_MODEL>`, or `<CAPTION_FONT>`, and pass them along when invoking `/label-sections` and `/add-captions`.

| Value | Override | Search order |
|---|---|---|
| `FFMPEG_DRAWTEXT` | `$FFMPEG_DRAWTEXT` | `ffmpeg` on PATH, then `<brew prefix>/opt/ffmpeg-full/bin/ffmpeg` — first one with drawtext |
| `WHISPER_BIN` | `$WHISPER_BIN` | `whisper-cli`, `whisper-cpp`, `main` on PATH |
| `WHISPER_MODEL` | `$WHISPER_MODEL` | `~/.cache/whisper-cpp/`, `~/whisper.cpp/models/`, `<brew prefix>/share/whisper-cpp/models/`, `<brew prefix>/share/whisper.cpp/models/` |
| `CAPTION_FONT` | `$CAPTION_FONT` | fontconfig (`fc-list`): a static Bold of "Big Shoulders Display", else of the newer merged "Big Shoulders" family |

Brew prefixes searched: the active `brew --prefix`, then `/opt/homebrew`, `/usr/local`, `/home/linuxbrew/.linuxbrew`.

```python
import os, shutil, subprocess

def brew_prefixes():
    # Apple Silicon, Intel Mac, and Linuxbrew defaults, with the active prefix first
    prefixes = ["/opt/homebrew", "/usr/local", "/home/linuxbrew/.linuxbrew"]
    if shutil.which("brew"):
        active = subprocess.run(["brew", "--prefix"], capture_output=True, text=True).stdout.strip()
        if active:
            prefixes = [active] + [p for p in prefixes if p != active]
    return prefixes

def has_drawtext(ffmpeg):
    out = subprocess.run([ffmpeg, "-hide_banner", "-filters"], capture_output=True, text=True).stdout
    return "drawtext" in out

def find_ffmpeg_drawtext():
    # $FFMPEG_DRAWTEXT, then ffmpeg on PATH (ffmpeg 9+ and most Linux builds include drawtext),
    # then Homebrew's keg-only ffmpeg-full
    candidates = [os.environ.get("FFMPEG_DRAWTEXT"), shutil.which("ffmpeg")]
    candidates += [f"{p}/opt/ffmpeg-full/bin/ffmpeg" for p in brew_prefixes()]
    return next((c for c in candidates if c and os.path.isfile(c) and has_drawtext(c)), None)

def find_whisper_bin():
    names = [os.environ.get("WHISPER_BIN"), "whisper-cli", "whisper-cpp", "main"]
    return next((shutil.which(n) for n in names if n and shutil.which(n)), None)

def find_whisper_model(name="ggml-medium.bin"):
    # $WHISPER_MODEL, then the user cache, then Homebrew share dirs (formula was renamed whisper-cpp -> whisper.cpp)
    candidates = [os.environ.get("WHISPER_MODEL"),
                  os.path.expanduser(f"~/.cache/whisper-cpp/{name}"),
                  os.path.expanduser(f"~/whisper.cpp/models/{name}")]
    for p in brew_prefixes():
        candidates += [f"{p}/share/whisper-cpp/models/{name}", f"{p}/share/whisper.cpp/models/{name}"]
    return next((c for c in candidates if c and os.path.isfile(c)), None)

def find_caption_font():
    # $CAPTION_FONT, else ask fontconfig for a static Bold of "Big Shoulders Display"
    # (preferred) or the newer merged "Big Shoulders" family. Variable fonts are skipped —
    # drawtext renders their thin default weight.
    env = os.environ.get("CAPTION_FONT")
    if env:
        env = os.path.expanduser(env)
        return env if os.path.isfile(env) else None
    if not shutil.which("fc-list"):
        return None
    out = subprocess.run(["fc-list", "--format", "%{file}|%{family}|%{style}|%{variable}\n"],
                         capture_output=True, text=True).stdout
    rows = [line.split("|") for line in out.splitlines()]
    rows = [parts for parts in rows if len(parts) == 4]
    # fontconfig also lists each named instance of a variable font ("Bold", "Black", ...)
    # as a non-variable entry, so skip every file that has a variable entry at all
    variable_files = {parts[0] for parts in rows if parts[3] == "True"}
    matches = {}
    for parts in rows:
        if parts[0] in variable_files or "Bold" not in parts[2].split(","):
            continue
        for rank, family in enumerate(["Big Shoulders Display", "Big Shoulders"]):
            if family in parts[1].split(","):
                matches.setdefault(rank, parts[0])
    return matches[min(matches)] if matches else None
```

The same functions appear in `/label-sections` and `/add-captions` so those skills work standalone — keep all three copies in sync.

### Pre-flight check

Before running any pipeline step, validate that all required tools and files are present. **Stop immediately and report what's missing** if any check fails.

```python
import sys

errors = []

# 1. ffmpeg / ffprobe
for tool in ["ffmpeg", "ffprobe"]:
    if not shutil.which(tool):
        errors.append(f"{tool} not found. Install with: brew install ffmpeg (macOS) or your package manager")

# 2. whisper-cli
WHISPER_BIN = find_whisper_bin()
if not WHISPER_BIN:
    errors.append("whisper-cli not found. Install with: brew install whisper-cpp (or build whisper.cpp and set $WHISPER_BIN)")

# 3. Whisper model
WHISPER_MODEL = find_whisper_model()
if not WHISPER_MODEL:
    errors.append("Whisper model ggml-medium.bin not found. Download it with: "
                  "mkdir -p ~/.cache/whisper-cpp && curl -L -o ~/.cache/whisper-cpp/ggml-medium.bin "
                  "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-medium.bin "
                  "(or set $WHISPER_MODEL to an existing model file)")

# 4. OpenCV with the Haar cascade face detector
try:
    subprocess.run([sys.executable, "-c",
        "import cv2, os; assert os.path.exists(cv2.data.haarcascades + 'haarcascade_frontalface_default.xml')"],
        capture_output=True, check=True)
except subprocess.CalledProcessError:
    errors.append("OpenCV 4.x with Haar cascades not found. Install with: pip3 install 'opencv-python-headless<5'")

# 5. SciPy (only when secondaries are provided)
if secondaries:
    try:
        subprocess.run([sys.executable, "-c", "import scipy"], capture_output=True, check=True)
    except subprocess.CalledProcessError:
        errors.append("scipy not found (required for multi-angle sync). Install with: pip3 install scipy numpy")

# 6. Captions prerequisites (only if captions are enabled)
if not nocaptions:
    FFMPEG_DRAWTEXT = find_ffmpeg_drawtext()
    if not FFMPEG_DRAWTEXT:
        errors.append("No ffmpeg with the drawtext filter found. Install with: brew install ffmpeg-full "
                      "(or set $FFMPEG_DRAWTEXT to a drawtext-capable ffmpeg)")

    CAPTION_FONT = find_caption_font()
    if not CAPTION_FONT:
        errors.append("Static Big Shoulders Display Bold font not found. Install the Bold TTF from "
                      "https://fonts.google.com/specimen/Big+Shoulders+Display or "
                      "https://github.com/xotypeco/big_shoulders (or set $CAPTION_FONT to the .ttf path)")
    # Verify it is a real font file (a bad download can save an HTML page as .ttf)
    elif "Big Shoulders" not in subprocess.run(["fc-scan", "--format", "%{family}", CAPTION_FONT], capture_output=True, text=True).stdout:
        errors.append(f"{CAPTION_FONT} is not a valid Big Shoulders font file.")

if errors:
    print("Pre-flight check failed:")
    for e in errors:
        print(f"  - {e}")
    # STOP — do not proceed with the pipeline
else:
    print(f"WHISPER_BIN={WHISPER_BIN}\nWHISPER_MODEL={WHISPER_MODEL}")
    if not nocaptions:
        print(f"FFMPEG_DRAWTEXT={FFMPEG_DRAWTEXT}\nCAPTION_FONT={CAPTION_FONT}")
```

### Pipeline

Run these steps in order. Each step takes the output of the previous step as input. All intermediate files go in the same directory as the input file.

Given input `<name>.<ext>` and optional secondaries:

#### Step 0 (multi-angle only): Sync Feeds (`/sync-feeds`)

Only run this step when secondary video files are provided.

- **Input**: `<name>.<ext>` (primary) + secondary video files
- **Output**: `<name>_synced.mp4` (trimmed primary) + `<name>_sync_manifest.json`
- Cross-correlate audio to find sync offsets for each secondary
- Trim only the primary to the common overlap range
- Write a sync manifest JSON with secondary file paths, offsets, and overlap info
- Secondaries are NOT trimmed — they stay as original files
- **From this point forward, use `<name>_synced.mp4` as the input** (replacing the raw primary)

#### Step 1: Remove Silence (`/remove-silence`)
- **Input**: `<name>_synced.mp4` (if multi-angle) or `<name>.<ext>` (if single camera)
- **Output**: `<name>_trimmed.mp4` + `<name>_segment_map.json`
- Detect silences > 0.5s using `silencedetect=noise=-30dB:d=0.5`
- Cut silences down to 0.3s natural pauses
- Join segments with re-encoding using trim/atrim filters (not stream copy — stream copy causes keyframe drift in the segment map and audio blips at boundaries)
- Write a segment map JSON recording which time ranges were kept (for downstream timestamp mapping)
- Note: Only the primary is processed. Secondaries are not trimmed.

#### Step 2: Label Sections (`/label-sections`)
- **Input**: `<name>_trimmed.mp4`
- **Output**: `<name>_trimmed_sections.json`
- Transcribe with `<WHISPER_BIN> -m <WHISPER_MODEL>`
- Break into sections: min 3s, max 6s per section
- Label each section as normal (1.0x), emphasis (1.25x), or critical (1.6x) based on content analysis
- ~60% normal, ~30% emphasis, ~10% critical (by duration — critical only for the one or two real punchlines)

#### Step 2.5 (multi-angle only): Swap Angles (`/swap-angles`)

Only run this step when secondary video files were provided.

- **Input**: `<name>_trimmed_sections.json` + `<name>_sync_manifest.json`
- **Output**: `<name>_trimmed_sections.json` (updated in place with `source` fields)
- Mark every 3rd normal section for a secondary camera angle
- Add source fields to the sections JSON for produce-zoom to use

#### Step 3: Produce Zoom (`/produce-zoom`)
- **Input**: `<name>_trimmed.mp4` + `<name>_trimmed_sections.json`
- **Multi-angle additional inputs**: `<name>_sync_manifest.json` + `<name>_segment_map.json`
- **Output**: `<name>_zoomed.mp4`
- **Resolution**: Pass the resolution flag (`-HD`, `-4K`, or `-portrait`) to this step.
- Detect face position per section using OpenCV (samples every 0.5s, size-filtered, median per section)
- For landscape: crop to zoom level centered on face, scale back to original resolution
- For `-portrait`: crop a 9:16 region from the 16:9 source centered on face. Normal = full frame (100% height), emphasis = 92% height, critical = 85% height (locked — tighter reads as an uncomfortable face crop). The crop never cuts off the top of the head; it keeps the head in frame and loses chest/shoulders as it zooms.
- For multi-angle: read the sync manifest and segment map to compute secondary file timestamps. Secondary sections use original (untrimmed) secondary files with mapped timestamps.
- Render with libx264 CRF 18

#### Step 4: Correct Colors (`/correct-colors`)
- **Input**: `<name>_zoomed.mp4`
- **Output**: `<name>_colorcorrected.mp4`
- **Resolution**: Pass the resolution flag to this step. For `-portrait`, output is already 1080x1920 from step 3.
- Apply: `colorbalance=rs=0.02:gs=-0.01:bs=-0.02,curves=m='0/0 0.25/0.20 0.75/0.82 1/1',eq=brightness=0.02:contrast=1.05:saturation=1.05`
- Video re-encoded, audio copied

#### Step 5: Master Audio (`/master-audio`)
- **Input**: `<name>_colorcorrected.mp4`
- **Output**: `<name>_mastered.mp4`
- Apply: `highpass=f=80,lowpass=f=14000,equalizer=f=3000:t=q:w=1.5:g=3,equalizer=f=200:t=q:w=1.0:g=2,acompressor=threshold=-21dB:ratio=3:attack=5:release=50,volume=0.6,loudnorm=I=-16:TP=-1.5:LRA=11`
- Video copied, audio re-encoded AAC 192k

#### Step 6: Add Captions (`/add-captions`)

**Skip this step if `-nocaptions` flag was provided.** When skipping, copy or rename `<name>_mastered.<ext>` to `<name>_final.mp4` instead:
```bash
ffmpeg -y -i <name>_mastered.<ext> -c copy <name>_final.mp4
```

If captions are enabled (the default):
- **Input**: `<name>_mastered.mp4`
- **Output**: `<name>_final.mp4`
- **Resolution**: Pass the resolution flag (`-HD`, `-4K`, or `-portrait`) to this step. Default is `-HD` (1920x1080).
- Transcribe with `<WHISPER_BIN> -m <WHISPER_MODEL>`
- For landscape: break into max 6 words per caption, ALL CAPS
- For `-portrait`: break into max 3 words per caption (narrower frame), ALL CAPS
- Rejoin contractions whisper split across segments, fix mis-heard names (same list as the transcript), keep apostrophes as ’, and stack captions over 4 words into balanced lines
- Big Shoulders Display Bold 700 via `fontfile=<CAPTION_FONT>` (NOT `font='Big Shoulders Display'`, which renders a thin weight), white text on black box (70% opacity)
- Font size calculated from **target** dimensions (not source): `target_width * 0.0495`, centered at `target_height * 0.80`
- For `-portrait`: target is 1080x1920, font size from `target_width * 0.065` (larger relative to narrow frame), centered at `target_height * 0.75` (higher to avoid thumb zone)
- If downscaling, prepend appropriate `scale=` filter to the filter chain before drawtext filters
- Renders with `<FFMPEG_DRAWTEXT>` (the drawtext-capable ffmpeg found in pre-flight)
- Output as `_final.mp4` (always mp4 regardless of input format)

#### Step 6.5: Save Transcript
- **Input**: the whisper transcript from Step 2 (taken from `<name>_trimmed.mp4`, so it matches the words in the final video)
- **Output**: `<original_filename_without_extension>.md` next to the original video. Use the original file's exact name and casing (e.g. `CollegeGradAdvice.mp4` → `CollegeGradAdvice.md`).
- Purpose: a clean, readable transcript the user can paste into social posts or hand to claude.ai. It is a deliverable, not an intermediate — cleanup never deletes it.
- Format:
  ```markdown
  # <Readable title from the filename, e.g. "College Grad Advice">

  - **Video:** `<name>_final.mp4`
  - **Length:** <m:ss of the final video>

  <transcript as paragraphs>
  ```
- Cleanup rules (light touch — it should still read as the speaker's own words):
  - Plain paragraphs, no timestamps. Break at changes of idea (roughly 2–5 sentences each).
  - Fix punctuation, capitalization and sentence boundaries that whisper got wrong.
  - Fix mis-heard names and products (e.g. "chat GBT" → "ChatGPT", "digital advisors" → "Digital Advisors", "digitaladvisors.ai").
  - Drop stutters and immediate word repeats ("this has, this has" → "this has"), but do not rephrase, summarize or add anything.
- If a `<name>.md` already exists, overwrite it (it was produced by a previous run of this pipeline).

### Output

The final file is `<name>_final.mp4` in the same directory as the input, along with the transcript `<name>.md`.

After completion, report:
- Original duration vs final duration
- Time saved from silence removal
- Number of zoom sections and label distribution
- Whether captions were included or skipped
- If multi-angle: number of secondary sections swapped, sync confidence
- Confirm all steps completed

#### Step 7: Review Final Output
- Open the final video for the user to review: `open <name>_final.mp4`
- Report the pipeline summary (duration, time saved, zoom sections, captions status)
- Ask the user to confirm they're happy with the result
- If they approve, proceed to Step 8 (cleanup)
- If they want changes, stop here — all intermediate files are preserved so individual steps can be re-run

#### Step 8: Clean Artifacts (`/clean-artifacts`)
- Delete intermediate files: `_synced`, `_sync_manifest.json`, `_segment_map.json`, `_trimmed`, `_trimmed_sections.json`, `_zoomed`, `_colorcorrected`, `_mastered`, `_captioned`
- Keep the original `<name>.<ext>`, all secondary camera originals, `<name>_final.mp4`, and the transcript `<name>.md`
- Report number of files deleted and disk space recovered
- No confirmation needed when called from this pipeline

### Error handling

If any step fails, stop the pipeline and report which step failed and why. Do not continue to subsequent steps. Do not run cleanup if the pipeline failed.
