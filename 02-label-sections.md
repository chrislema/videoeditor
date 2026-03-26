---
name: label-sections
description: Analyze video transcript and label sections as normal/emphasis/critical for dynamic zoom editing
user_invocable: true
metadata:
  tags: video, editing, zoom, transcript, content-analysis
---

## When to use

Use this skill when the user wants to break a video transcript into labeled zoom sections for dynamic cropping/zoom editing. Triggered by requests like "label sections," "zoom sections," "break into sections," "dynamic zoom," or "label-zoom-sections."

## How to use

### Prerequisites
- `ffmpeg` and `whisper-cli` must be installed
- A whisper model must be available at `/opt/homebrew/share/whisper-cpp/models/ggml-medium.bin`
- Input must be a video file (mp4, mov, mkv, webm, etc.)

### Parameters
The user may optionally specify:
- **input**: Path to the video file (if not provided, ask)
- **sections_per_minute**: Target number of sections per minute (default: 20)
- **min_section_duration**: Minimum length of any section in seconds (default: 3)
- **max_section_duration**: Maximum length of any section in seconds (default: 6)
- **zoom_levels**: Override zoom multipliers (defaults: normal=1.0, emphasis=1.25, critical=1.6)

### Process

#### Step 1: Transcribe the video

Extract audio and run whisper-cli to get timestamped transcript:

```bash
ffmpeg -i <input> -ar 16000 -ac 1 -f wav /tmp/zoom_whisper_$$.wav -y
whisper-cli -m /opt/homebrew/share/whisper-cpp/models/ggml-medium.bin -f /tmp/zoom_whisper_$$.wav
rm -f /tmp/zoom_whisper_$$.wav
```

Parse the `[start --> end] text` lines from output.

#### Step 2: Segment the transcript

Break the transcript into sections following these rules:
- Target ~20 sections per minute (adjustable)
- **Minimum 3 seconds per section** — sections shorter than 3 seconds look manic. Merge short sections with their neighbor (prefer merging with the adjacent section that has the same or closest label)
- **Maximum 6 seconds per section** — split longer segments at natural phrase boundaries
- Split on natural phrase/sentence boundaries from the whisper timestamps
- Whisper segments that are too long should be subdivided at logical phrase breaks
- When merging short sections, the merged section takes the label of the more important content (critical > emphasis > normal)

#### Step 3: Label each section

Analyze the **content and rhetorical function** of each section and assign a label:

**normal** (baseline zoom, 100%)
- Narrative setup and scene-setting
- Background information and context
- Transitions between ideas
- Asides, caveats, qualifications ("I read it in a book," "whether it was research or fabrication")
- Connective tissue ("And so," "And what tends to happen")

**emphasis** (medium zoom, 125%)
- Key supporting points that build toward a conclusion
- Rhetorical questions that create tension ("then what do you do next?")
- Contrasts and comparisons being set up
- Moments of rising energy or pace
- Repetition for effect ("better and better and better")
- Important details that support the argument

**critical** (tight zoom, 160%)
- Core thesis statements and main takeaways
- Punchlines and reveals ("quantity, not quality side")
- Emotional peaks and moments of conviction
- Statements the speaker would want clipped for social media
- The "so what" moments — why this matters
- Surprising or counterintuitive claims
- Calls to action or challenge statements

#### Labeling guidelines

- The **first half** of most talks is setup — lean toward normal/emphasis
- The **second half** typically has the payoff — more emphasis/critical
- **Don't over-index on critical** — if everything is critical, nothing is. Aim for roughly 40% normal, 35% emphasis, 25% critical
- **No more than 2 consecutive sections at the same zoom level.** If 3+ sections in a row would have the same label/zoom, toggle the middle one to an adjacent level (e.g., flip the 3rd consecutive emphasis to normal, or the 3rd consecutive normal to emphasis). This prevents long visual stretches with no zoom change — even a 4-second emphasis between two normals creates a noticeable visual beat.
- When in doubt between two labels, consider: "Would this moment work as a standalone clip?" If yes, it's critical. If it needs context, it's emphasis or normal.

### Output format

Write a JSON file named `<input_basename>_sections.json` with this structure:

```json
{
  "metadata": {
    "source": "<filename>",
    "duration": <total_seconds>,
    "zoom_levels": {
      "normal": 1.0,
      "emphasis": 1.25,
      "critical": 1.6
    },
    "total_sections": <count>
  },
  "sections": [
    {
      "start": 0.000,
      "end": 2.720,
      "label": "normal",
      "text": "You've likely heard the story, right?"
    }
  ]
}
```

### Important notes

- Timestamps must be continuous — each section's start must equal the previous section's end
- The first section must start at 0.000
- The last section must end at the video's total duration
- The `text` field is for reference/debugging — it shows what's being said in that section
- This JSON is consumed by the zoom/crop rendering pipeline (face detection + dynamic crop)
