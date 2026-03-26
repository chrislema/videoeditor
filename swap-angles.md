---
name: swap-angles
description: Mark every 3rd normal section for a secondary camera angle to create visual variety in multi-angle edits
user_invocable: true
metadata:
  tags: video, multi-camera, editing, angles, cuts, multi-angle
---

## When to use

Use this skill after `/label-sections` when working with multi-angle footage. It reads the sections JSON and marks every 3rd normal section to be sourced from a secondary camera angle. Triggered by `/swap-angles` or requests like "add angle cuts," "swap in the other camera," or "mix in the second angle."

## How to use

### Prerequisites
- A sections JSON file (produced by `/label-sections`)
- One or two secondary video files (synced and trimmed to match the primary)

### Parameters
- **sections_json**: Path to the sections JSON file (if not provided, look for `*_sections.json` in the current directory)
- **secondaries**: Paths to 1-2 secondary video files. These must be synced and trimmed to the same duration as the primary.
- **swap_ratio**: How often to swap (default: every 3rd normal). The value N means "swap 1 out of every N normal sections." Default is 3.
- **start_at**: Which normal in the cycle to swap (default: 3, meaning the 3rd normal gets swapped, then the 6th, 9th, etc.)

Example invocations:
- `/swap-angles testvideo_trimmed_sections.json angle2_trimmed.mp4`
- `/swap-angles sections.json angle2.mp4 angle3.mp4`

### Process

#### Step 1: Load sections JSON

Read the sections JSON. It has this structure:

```json
{
  "metadata": {
    "source": "testvideo_trimmed.mp4",
    "duration": 203.6,
    "zoom_levels": {
      "normal": 1.0,
      "emphasis": 1.25,
      "critical": 1.6
    },
    "total_sections": 65
  },
  "sections": [
    { "start": 0.000, "end": 6.360, "label": "normal", "text": "..." },
    { "start": 6.360, "end": 10.200, "label": "emphasis", "text": "..." }
  ]
}
```

#### Step 2: Identify normal sections and mark swaps

```python
import json

def mark_swaps(sections, secondaries, swap_ratio=3, start_at=3):
    """Add a 'source' field to each section.

    - emphasis and critical sections: always 'primary'
    - normal sections: 'primary' unless it's the Nth normal in the cycle
    - Swapped normals round-robin through secondaries

    Returns the modified sections list and swap stats.
    """
    normal_count = 0
    swap_count = 0
    secondary_idx = 0  # for round-robin

    for section in sections:
        if section["label"] != "normal":
            section["source"] = "primary"
            continue

        normal_count += 1

        if normal_count % swap_ratio == 0:
            # This normal gets swapped
            section["source"] = f"secondary_{secondary_idx + 1}"
            secondary_idx = (secondary_idx + 1) % len(secondaries)
            swap_count += 1
        else:
            section["source"] = "primary"

    return sections, normal_count, swap_count
```

#### Step 3: Verify secondary durations

Before writing the output, verify that each secondary video is at least as long as the primary's duration (from the sections JSON metadata). If not, warn — some swapped sections may extend beyond the secondary's footage.

```python
import subprocess

def get_duration(path):
    result = subprocess.run(
        ["ffprobe", "-v", "error", "-show_entries", "format=duration",
         "-of", "default=noprint_wrappers=1:nokey=1", path],
        capture_output=True, text=True
    )
    return float(result.stdout.strip())

primary_duration = data["metadata"]["duration"]
for sec_path in secondaries:
    sec_dur = get_duration(sec_path)
    if sec_dur < primary_duration - 0.5:
        print(f"WARNING: {sec_path} ({sec_dur:.1f}s) is shorter than primary ({primary_duration:.1f}s)")
```

#### Step 4: Write updated sections JSON

Write the modified JSON back to the same file, adding secondary source info to the metadata.

```python
data["metadata"]["secondary_sources"] = secondaries  # list of file paths
data["metadata"]["swap_ratio"] = f"1:{swap_ratio}"
data["metadata"]["swapped_sections"] = swap_count

with open(sections_json_path, "w") as f:
    json.dump(data, f, indent=2)
```

### Output

The sections JSON file is updated **in place** with:
- A `"source"` field on every section (`"primary"`, `"secondary_1"`, or `"secondary_2"`)
- New metadata fields: `"secondary_sources"`, `"swap_ratio"`, `"swapped_sections"`

Example output:

```json
{
  "metadata": {
    "source": "testvideo_trimmed.mp4",
    "duration": 203.6,
    "zoom_levels": {
      "normal": 1.0,
      "emphasis": 1.25,
      "critical": 1.6
    },
    "total_sections": 65,
    "secondary_sources": ["angle2_trimmed.mp4"],
    "swap_ratio": "1:3",
    "swapped_sections": 9
  },
  "sections": [
    { "start": 0.000, "end": 4.200, "label": "normal", "text": "...", "source": "primary" },
    { "start": 4.200, "end": 8.400, "label": "emphasis", "text": "...", "source": "primary" },
    { "start": 8.400, "end": 12.000, "label": "normal", "text": "...", "source": "primary" },
    { "start": 12.000, "end": 16.500, "label": "normal", "text": "...", "source": "primary" },
    { "start": 16.500, "end": 20.800, "label": "normal", "text": "...", "source": "secondary_1" },
    { "start": 20.800, "end": 25.100, "label": "critical", "text": "...", "source": "primary" }
  ]
}
```

### Report

Print:
- Total sections and label distribution (normal / emphasis / critical)
- Number of normal sections swapped and to which secondary
- Swap pattern (e.g., "1 out of every 3 normals → secondary_1")
- If multiple secondaries, the round-robin assignment

### Important notes

- **Only normal sections are swapped.** Emphasis and critical always stay on the primary camera — they get the zoom treatment.
- **Swapped sections get no zoom.** The produce-zoom step will render secondary-sourced sections as full-frame, scaled to the output resolution. This provides visual variety through the angle change itself, not through a zoom effect.
- **Round-robin for multiple secondaries.** With two secondary cameras, the swaps alternate: 3rd normal → secondary_1, 6th normal → secondary_2, 9th normal → secondary_1, etc.
- **The swap ratio is adjustable** but defaults to 1:3 (every 3rd normal). Lower ratios (1:2) create more cuts. Higher ratios (1:4, 1:5) are more subtle.
- **Don't swap the very first normal.** Starting with the primary establishes the "home base" angle. The first swap should feel like a deliberate cut away, not an arbitrary start.
- **Secondary files must be synced.** The time ranges in the sections JSON correspond to the primary's timeline. Since the files were synced by `/sync-feeds`, the same timestamps work across all feeds.
