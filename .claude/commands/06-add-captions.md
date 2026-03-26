---
name: add-captions
description: Burn in bold all-caps captions with black background boxes using whisper transcription and ffmpeg
user_invocable: true
metadata:
  tags: video, captions, subtitles, ffmpeg, whisper, typography
---

## When to use

Use this skill when the user wants to add burned-in captions/subtitles to a video. Triggered by requests like "add captions," "burn in subtitles," "caption this video," or "add-captions."

## How to use

### Prerequisites
- `ffmpeg` must be installed with drawtext/libfreetype support (standard Homebrew `ffmpeg` includes this)
- `whisper-cli` must be installed with a model at `/opt/homebrew/share/whisper-cpp/models/ggml-medium.bin`
- Big Shoulders Display Bold 700 font installed at `~/Library/Fonts/BigShouldersDisplay-Bold.ttf`. The font is resolved via fontconfig by name (`font='Big Shoulders Display'`), not by file path. Always use `font=` instead of `fontfile=`.

### Parameters
The user may optionally specify:
- **input**: Path to the video file (if not provided, ask)
- **output**: Path for the output file. When called from `/process-video`, the pipeline passes `<name>_final.mp4` as the output path. When run standalone, default: strip `_mastered` from the input basename, then append `_final.mp4` — e.g., `mainvideo_mastered.mp4` → `mainvideo_final.mp4`.
- **resolution**: `-HD` (1920x1080), `-4K` (keep source), or `-portrait` (1080x1920). Default: `-HD`
- **max_words**: Maximum words per caption line (default: 6 for landscape, 3 for portrait)
- **target_width**: Target percentage of video width for text (default: 80%)
- **box_opacity**: Opacity of the black background box, 0.0-1.0 (default: 0.70, meaning 30% transparent)

### Process

#### Step 1: Get video dimensions

```python
probe = subprocess.run(["ffprobe", "-v", "error", "-select_streams", "v:0",
    "-show_entries", "stream=width,height", "-of", "csv=p=0", video],
    capture_output=True, text=True)
width, height = map(int, probe.stdout.strip().split(","))
```

#### Step 2: Transcribe with whisper

Use whisper's `-ml` (max segment length) flag to get short, accurately-timed segments directly from whisper. This is critical — do NOT split whisper segments manually and distribute time evenly, as speech is not evenly paced and captions will drift ahead of the audio.

```bash
ffmpeg -y -i <input> -ar 16000 -ac 1 -f wav /tmp/caption_whisper_$$.wav
# -ml 35 for landscape (~6 words), -ml 20 for portrait (~3 words)
whisper-cli -m /opt/homebrew/share/whisper-cpp/models/ggml-medium.bin -ml <max_chars> -f /tmp/caption_whisper_$$.wav
rm -f /tmp/caption_whisper_$$.wav
```

Where `<max_chars>` is **35** for landscape (roughly 6 words) or **20** for portrait (roughly 3 words).

Parse the `[HH:MM:SS.mmm --> HH:MM:SS.mmm] text` lines from output.

#### Step 3: Build caption events

- Each whisper segment becomes one caption event (no further splitting needed — whisper's timestamps are audio-accurate)
- Convert all text to UPPERCASE
- Each segment's start/end time comes directly from whisper

#### Step 4: Determine target dimensions and calculate font size

Set target dimensions based on the resolution flag:

```python
# Determine target dimensions
if resolution == "portrait":
    target_w, target_h = 1080, 1920
elif resolution == "4K":
    target_w, target_h = width, height
else:  # HD (default)
    target_w, target_h = 1920, 1080

# Calculate font size from TARGET dimensions
if resolution == "portrait":
    # Portrait: larger relative font for narrow frame, higher position to avoid thumb zone
    font_size = int(target_w * 0.065)   # ~70px at 1080px wide
    box_padding = int(font_size * 0.10)
    y_position = int(target_h * 0.75)   # higher up — avoids mobile UI/thumb zone at bottom
else:
    font_size = int(target_w * 0.0495)  # ~95px at 1920px, ~190px at 3840px
    box_padding = int(font_size * 0.07)
    y_position = int(target_h * 0.80)
```

#### Step 5: Build drawtext filter chain

Each caption event becomes a separate `drawtext` filter with `enable='between(t,start,end)'`:

```python
def escape_drawtext(text):
    text = text.replace("\\", "\\\\")
    text = text.replace("'", "\u2019")  # right single quote
    text = text.replace(":", "\\:")
    text = text.replace(",", "\\,")
    text = text.replace(";", "\\;")
    text = text.replace("[", "\\[")
    text = text.replace("]", "\\]")
    text = text.replace("%", "%%")
    return text

for start, end, text in caption_events:
    escaped = escape_drawtext(text)
    dt = (
        f"drawtext=font='Big Shoulders Display'"
        f":text='{escaped}'"
        f":fontcolor=white"
        f":fontsize={font_size}"
        f":box=1"
        f":boxcolor=black@0.70"
        f":boxborderw={box_padding}"
        f":x=(w-text_w)/2"
        f":y={y_position}"
        f":enable='between(t\\,{start:.3f}\\,{end:.3f})'"
    )
```

Chain all drawtext filters with commas, write to a PID-unique temp file (e.g., `/tmp/caption_filter_{os.getpid()}.txt`) to avoid collisions with concurrent runs.

#### Step 6: Build filter chain and render with ffmpeg

If scaling is needed, prepend the appropriate `scale=` filter to the filter chain before the drawtext filters:

```python
if resolution == "portrait":
    # Portrait video is already 1080x1920 from the zoom step, but verify
    if width != 1080 or height != 1920:
        filter_chain = f"scale=1080:1920,{drawtext_filters}"
    else:
        filter_chain = drawtext_filters
elif resolution != "4K" and (width != 1920 or height != 1080):
    filter_chain = f"scale=1920:1080,{drawtext_filters}"
else:
    filter_chain = drawtext_filters
```

Write the full filter chain to the temp file, then render:

```bash
ffmpeg -y -i <input> \
  -filter_complex_script /tmp/caption_filter_$$.txt \
  -c:v libx264 -preset fast -crf 18 \
  -c:a copy \
  <output>
```

**Important**: Use `font='Big Shoulders Display'` (fontconfig name resolution), NOT `fontfile=` (which may be silently ignored and fall back to Verdana). Standard Homebrew `ffmpeg` includes drawtext with libfreetype and libfontconfig — no separate `ffmpeg-full` tap is needed.

### Caption style

- **Font**: Big Shoulders Display, weight 700 (Bold)
- **Case**: ALL CAPS
- **Color**: White text
- **Background**: Black box at 70% opacity (30% transparent), sized to fit the text
- **Position**: Centered horizontally. Landscape: 80% of video height. Portrait: 75% of video height (higher to avoid mobile thumb zone).
- **Max words per line**: 6 (landscape) / 3 (portrait)
- **Target width**: ~80% of video width
- **Font size multiplier**: Landscape: `target_width * 0.0495`. Portrait: `target_width * 0.065` (larger relative to narrow frame).

### Adjustments

**Bigger/smaller text:**
- Adjust the `width * 0.0495` multiplier. Higher = bigger text.

**More/fewer words per line:**
- Change max_words. Fewer words = bigger text per word, more frequent changes.

**Box transparency:**
- `boxcolor=black@0.70` — the number after @ is opacity (1.0 = fully opaque, 0.0 = invisible)

**Position:**
- Adjust `y_position` — `height * 0.80` puts it in the lower fifth. Use `height * 0.85` for lower, `height * 0.70` for higher.

### Important notes
- This skill requires `ffmpeg` with drawtext/libfreetype/libfontconfig support (standard Homebrew `ffmpeg` includes this)
- The font is resolved via fontconfig by name (`font='Big Shoulders Display'`), NOT by file path — `fontfile=` is silently ignored when fontconfig is enabled and falls back to Verdana
- The font file `BigShouldersDisplay-Bold.ttf` must be installed at `~/Library/Fonts/` for fontconfig to find it
- The black box automatically sizes to fit the text — it is not a fixed-width bar
- This skill should run after all other video processing (silence removal, zoom, color, audio mastering)
