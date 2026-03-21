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
- `ffmpeg` must be installed with drawtext/libfreetype support (via `homebrew-ffmpeg/ffmpeg` tap)
- `whisper-cli` must be installed with a model at `/opt/homebrew/share/whisper-cpp/models/ggml-medium.bin`
- Big Shoulders Display Bold 700 font at `/Users/chrislema/Library/Fonts/BigShouldersDisplay-700.ttf`

### Parameters
The user may optionally specify:
- **input**: Path to the video file (if not provided, ask)
- **output**: Path for the output file (default: `<input_basename>_captioned.<ext>`)
- **max_words**: Maximum words per caption line (default: 6)
- **target_width**: Target percentage of video width for text (default: 70%)
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

```bash
ffmpeg -y -i <input> -ar 16000 -ac 1 -f wav /tmp/caption_whisper.wav
whisper-cli -m /opt/homebrew/share/whisper-cpp/models/ggml-medium.bin -f /tmp/caption_whisper.wav
```

Parse the `[HH:MM:SS.mmm --> HH:MM:SS.mmm] text` lines from output.

#### Step 3: Break into caption events

- Split each transcript segment into chunks of max 6 words
- Convert all text to UPPERCASE
- Distribute the segment's time evenly across its chunks
- Each chunk becomes a separate caption event with start/end time

#### Step 4: Calculate font size

For 70% width target with Big Shoulders Display Bold at max 6 words:

```python
font_size = int(width * 0.062)  # ~238px at 3840px width
box_padding = int(font_size * 0.07)
y_position = int(height * 0.80)
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
        f"drawtext=fontfile='{font_path}'"
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

Chain all drawtext filters with commas, write to a temp file (too long for command line).

#### Step 6: Render with ffmpeg

```bash
ffmpeg -y -i <input> \
  -/filter_complex /tmp/caption_filter.txt \
  -c:v libx264 -preset fast -crf 18 \
  -c:a copy \
  <output>
```

**Important**: The `ffmpeg` binary must have libfreetype/drawtext support. Install via `brew install homebrew-ffmpeg/ffmpeg/ffmpeg --with-fdk-aac` (the standard Homebrew `ffmpeg` formula lacks drawtext).

### Caption style

- **Font**: Big Shoulders Display, weight 700 (Bold)
- **Case**: ALL CAPS
- **Color**: White text
- **Background**: Black box at 70% opacity (30% transparent), sized to fit the text
- **Position**: Centered horizontally, at 80% of video height
- **Max words per line**: 6
- **Target width**: ~70% of video width

### Adjustments

**Bigger/smaller text:**
- Adjust the `width * 0.062` multiplier. Higher = bigger text.

**More/fewer words per line:**
- Change max_words. Fewer words = bigger text per word, more frequent changes.

**Box transparency:**
- `boxcolor=black@0.70` — the number after @ is opacity (1.0 = fully opaque, 0.0 = invisible)

**Position:**
- Adjust `y_position` — `height * 0.80` puts it in the lower fifth. Use `height * 0.85` for lower, `height * 0.70` for higher.

### Important notes
- This skill requires `ffmpeg` with drawtext/libfreetype support (installed via `homebrew-ffmpeg/ffmpeg` tap)
- The font file `BigShouldersDisplay-700.ttf` is a static weight 700 instance downloaded from Google Fonts CDN
- The black box automatically sizes to fit the text — it is not a fixed-width bar
- This skill should run after all other video processing (silence removal, zoom, color, audio mastering)
