---
name: clean-artifacts
description: Remove intermediate video files from the pipeline, keeping only the original and final output
user_invocable: true
metadata:
  tags: video, cleanup, pipeline, files
---

## When to use

Use this skill after the video pipeline completes to clean up intermediate files. Triggered by requests like "clean up," "remove intermediates," "clean artifacts," or automatically at the end of `/process-video`.

## How to use

### Input

The base filename used in the pipeline. If not provided, infer from the `_final.mp4` file in the current directory.

### Process

Given a base name `<name>`, delete these intermediate files if they exist:

- `<name>_trimmed.mp4` (from step 1: remove-silence)
- `<name>_trimmed_sections.json` (from step 2: label-sections)
- `<name>_zoomed.mp4` (from step 3: produce-zoom)
- `<name>_colorcorrected.mp4` (from step 4: correct-colors)
- `<name>_mastered.mp4` (from step 5: master-audio)
- `<name>_captioned.mp4` (from step 6, if different from final)

### Keep

- `<name>.<ext>` — the original raw video
- `<name>_final.mp4` — the finished output

### Confirmation

Before deleting, list the files that will be removed and their total size. Ask the user to confirm unless this was called automatically from `/process-video`.

When called from `/process-video`, proceed without asking — the user already opted into the full pipeline.

### Report

After cleanup, report:
- Number of files deleted
- Disk space recovered
- Files remaining in directory
