# Sora P1 - Dubbing Pipeline Guide

## Voice Settings (FIXED - do not change)

| Character | Actor | TTS Voice | Rate | Words/sec |
|-----------|-------|-----------|------|-----------|
| Charlie (woman) | Charlie Chan | en-GB-SoniaNeural | +20% | ~3 w/s |
| Parker (man) | Parker Leung | en-GB-RyanNeural | +20% | ~3 w/s |

### TTS Commands
```bash
# Charlie
edge-tts --voice en-GB-SoniaNeural --rate "+20%" --text "..." --write-media seg_XXX.mp3

# Parker
edge-tts --voice en-GB-RyanNeural --rate "+20%" --text "..." --write-media seg_XXX.mp3
```

## Pipeline Steps (for future parts)

### 1. Generate TTS audio
```bash
edge-tts --voice <VOICE> --rate "+20%" --text "<YOUR_TEXT>" --write-media segments_raw_v3/seg_NNN.mp3
ffmpeg -y -i segments_raw_v3/seg_NNN.mp3 -ac 1 -ar 24000 segments_wav_v3/seg_NNN.wav
```

### 2. Detect speech segments from original video (Whisper)
```bash
ffmpeg -y -i <clip>.mp4 -ac 1 -ar 16000 /tmp/whisper_<clip>.wav
whisper /tmp/whisper_<clip>.wav --model tiny --language en --word_timestamps True --output_format json
# Save JSON, then generate per-clip SRT at /tmp/clip_srt/clip_<name>.srt
```

### 3. Mix TTS into clip (speed-adjusted to whisper slots)
- Use atempo to fit TTS into the whisper-detected speech duration
- Factor = tts_duration / slot_duration (no headroom)
- Clamp factor to [0.5, 2.0]
- Place into mix buffer at slot start sample

### 4. Replace audio in MP4
```bash
ffmpeg -y -i <original_clip>.mp4 -i clips_processed_v4/<clip>_new.wav -c:v copy -c:a aac -b:a 192k -ar 48000 -ac 2 -map 0:v:0 -map 1:a:0 -shortest output_v4/<clip>_new.mp4
```

### 5. Concatenate all clips in order
```bash
ffmpeg -y -f concat -safe 0 -i concat.txt -c copy mergeX.mp4
```

### 6. Generate subtitles from docx script
- Read cleaned script (no "The man/woman says:" prefixes, no "||")
- Map each script line to its clip's whisper slot timings
- Fill gaps between subtitles
- Save as .srt

### 7. Burn subtitles into video (optional)
Use Python PIL to render subtitle PNGs, overlay with ffmpeg.

## Clip Ordering
1, 2, 3_slow, 4a, 5, 6_edited, 7_edited, 8_edited, 9_edited, 10_edited

## Important Notes
- Script in `Video_Generator_Script_clean.docx` has 11 lines (no extra lines)
- "Parker Leung" in script is pronounced "Parker Learn" by TTS (correct)
- Clip 4a = clip 4 trimmed by 1s at end (best lip-sync choice)
- All TTS pre-generated at +20% rate to reduce atempo stretching
- Use whisper (tiny model) on original clip audio for speech segment detection
- No headroom (slot_dur * 0.9) — use full slot duration for target
