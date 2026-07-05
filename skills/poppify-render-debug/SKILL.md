---
name: poppify-render-debug
description: Verify a rendered Poppify reel by downloading the MP4, running ffprobe, and extracting sample frames. Use after confirm() returns videoUrl, OR when the user says: "check the video", "verify the reel", "is the render OK", "the video looks wrong", "missing audio", "no music", "no sound", "captions missing", "captions wrong color", "text is wrong color", "wrong duration", "video is too short", "video is too long", "no captions on slide N", or any similar verification / debugging request on a finished Poppify render. Produces a Markdown verdict table.
---

# Verifying a Poppify render

Once `get_result({ sessionId, apiKey })` returns `status: "complete"` with a `videoUrl`, run this verification BEFORE handing the URL to the user. The signed GCS URL is valid ~7 days (`videoUrlExpiresAtIso` has the exact expiry), so verify, then hand it over with the expiry note.

## Prerequisites

```bash
which ffprobe ffmpeg curl   # all three should exist
```

If ffmpeg/ffprobe are missing: `brew install ffmpeg` on macOS, `apt-get install ffmpeg` on Debian/Ubuntu.

## Step 1 — Download the MP4

```bash
mkdir -p /tmp/poppify-verify
curl -sL "$VIDEO_URL" -o /tmp/poppify-verify/render.mp4
ls -l /tmp/poppify-verify/render.mp4   # should be ≥ 200KB for a 15s reel
```

If the file is < 50KB, the URL is likely expired or the render produced an empty file — re-call `get_result` to confirm `status` is still `complete` and `videoUrl` is current.

## Step 2 — ffprobe for the structural verdict

```bash
ffprobe -v error -print_format json -show_streams -show_format /tmp/poppify-verify/render.mp4
```

Check these fields against expectations:

| Question | Where to look | Expected |
|---|---|---|
| Has video? | `streams[].codec_type === "video"` | h264, 720x1280, ~24fps |
| Has audio? | `streams[].codec_type === "audio"` | aac, sample_rate 44100 or 48000 |
| Duration | `format.duration` | matches the sum of per-slide durations from `get_slide_plan` / `get_result` (text-length-driven; media floors play in full) within ±1s |
| Resolution | `streams[0].width` x `streams[0].height` | 720x1280 (vertical) |

If audio stream is missing: the session's `selectedAudio` likely had an empty `resolvedUrl`. Check `get_result`'s response for the session's audio attach state.

## Step 3 — Extract sample frames

Extract one frame per second to a thumbs directory:

```bash
ffmpeg -y -i /tmp/poppify-verify/render.mp4 -vf "fps=1" /tmp/poppify-verify/frame_%03d.png 2>/dev/null
ls /tmp/poppify-verify/frame_*.png
```

Read each frame with the Read tool (it accepts PNG paths) and verify visually:

- **Caption present?** Text should appear on most slides (recipe controls which beats get captions). Empty caption on a slide is a deliberate `set_text({newText:""})` patch — not a bug.
- **Caption color matches request?** If user asked for `textColor: "#FFFFFF"` and you see another color, the schema-drop bug returned — file via `submit_feedback`.
- **Caption position matches `textAnimation`?** typewriter → upper third (~33% height), bold_captions → middle, phrase_reveal → lower third (block bottom ~84%).
- **Image content matches slides[i] intent?** If a slide's `voiceoverShort` says "before" but the image is the "after" frame, the visual edits queue may have applied in wrong order.

## Step 4 — Produce a verdict for the user

Output a small Markdown table:

```markdown
| Check | Result |
|---|---|
| Video stream | ✅ h264 720x1280 @ 24fps |
| Audio stream | ✅ aac 44.1kHz, 30.0s |
| Duration | ✅ 30.2s, matches per-slide sum (29.8s) |
| Caption color | ✅ matches request (#FFFFFF) |
| Caption position | ✅ lower third (phrase_reveal) |
| All slides have captions | ⚠️ slide 3 has no caption (intentional? check session.slides[3].voiceoverShort) |
```

If everything is green, hand the URL to the user with a note that it expires in ~7 days (save it before then). If anything is red, surface the specific failure and offer to re-render with a corrected patch.

## Common failure patterns

- **Audio stream missing.** Library audio attached without resolution OR the audio was attached with an `assetId` that's no longer in the library. Resolution: call `get_music_library` for a fresh asset list, re-attach via `apply_session_patch({audio:{source:"library", assetId}})`, then `confirm` again.
- **Duration drift > 1s.** Usually means slides ended up with very short `voiceoverShort` text and no explicit duration, so the text-driven length computed below expectations. Resolution: write longer text (`update_slides set_text`), OR set an explicit per-slide hold via `update_slides({action:"set_duration", slideIndex, duration:N})` (2–15s, any slide, overrides text-length). Note: voiceover audio and rendered live-motion clips are media floors that always play in full — they don't drift, so a short live-clip render points at a truncation bug, not text length.
- **Wrong caption color.** If `textColor` was requested but not applied, the MCP `apply_session_patch` schema may be stale. Re-check current build via `poppify-schema-introspect` skill.
- **Captions baked into the image AND drawn by composer (double text).** Don't render text-heavy slides via `generate_image` — the model bakes text into the image AND composer adds drawtext on top. Use HTML/CSS screencap + `upload_asset` instead, and pass `update_slides({action:"set_text",newText:""})` to skip composer text for that slide.

## Cleanup

```bash
rm -rf /tmp/poppify-verify
```
