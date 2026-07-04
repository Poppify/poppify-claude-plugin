---
name: poppify-render-debug
description: OPTIONAL deep-verify of a finished Poppify render — downloads the MP4, runs ffprobe, and extracts sample frames. Requires a local shell with ffmpeg + ffprobe + curl (available in Claude Code; NOT in shell-less clients like Claude Desktop). Reach for it when the user reports a problem ("the video looks wrong", "missing audio", "no music", "captions missing", "wrong color text", "wrong duration") or explicitly asks to verify the reel. It is never a required step — a clean render does not need it.
---

# Verifying a Poppify render (optional)

This is an **optional** diagnostic, not a mandatory post-render step. Most renders are fine and can be handed straight to the user. Use it when the user reports something wrong, or explicitly asks you to check the file.

## Capability gate — check BEFORE attempting

This skill needs a local shell with `ffmpeg`, `ffprobe`, and `curl`. Many MCP clients (Claude Desktop, web, most non-CLI hosts) have **no shell tool at all** — there is nothing to run these in.

```bash
which ffprobe ffmpeg curl   # all three must exist to proceed
```

- **If you have no Bash/shell tool, or the check fails:** SKIP this skill. Do NOT tell the user to install anything. Just hand them the `videoUrl` and suggest they watch it; if they report a specific issue, fall back to the `poppify-troubleshoot` decision tree (which is pure MCP-tool reasoning, no shell needed).
- **If ffprobe/ffmpeg are missing but you DO have a shell:** offer — don't insist — that they can `brew install ffmpeg` (macOS) / `apt-get install ffmpeg` (Debian/Ubuntu) if they want the deep check. Otherwise skip.
- **Only if all three exist:** proceed below.

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
| Duration | `format.duration` | matches session.duration (`15s` / `30s` / `60s`) within ±0.5s |
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
- **Caption position matches `textAnimation`?** typewriter → top-center, bold_captions → middle, phrase_reveal → bottom-left.
- **Image content matches slides[i] intent?** If a slide's `voiceoverShort` says "before" but the image is the "after" frame, the visual edits queue may have applied in wrong order.

## Step 4 — Produce a verdict for the user

Output a small Markdown table:

```markdown
| Check | Result |
|---|---|
| Video stream | ✅ h264 720x1280 @ 24fps |
| Audio stream | ✅ aac 44.1kHz, 30.0s |
| Duration | ✅ matches session.duration=30s |
| Caption color | ✅ matches request (#FFFFFF) |
| Caption position | ✅ bottom-left (phrase_reveal) |
| All slides have captions | ⚠️ slide 3 has no caption (intentional? check session.slides[3].voiceoverShort) |
```

If everything is green, hand the URL to the user (the signed URL is valid ~7 days). If anything is red, surface the specific failure and offer to re-render with a corrected patch.

## Common failure patterns

- **Audio stream missing.** Library audio attached without resolution OR `apply_session_patch({audio:...})` was called with an `assetId` that's no longer in the library. Resolution: call `get_music_library` for a fresh asset list, then `apply_session_patch({audio:{source:"library", assetId}})` again, then `confirm` again.
- **Duration drift > 1s.** Usually means slides ended up with very short `voiceoverShort` text and the renderer's text-driven duration computed below the recipe minimum. Resolution: write longer text on slides via `update_slides set_text`.
- **Wrong caption color.** If `textColor` was requested but not applied, the MCP apply_session_patch schema may be stale. Re-check current build via `poppify-schema-introspect` skill.
- **Captions baked into the image AND drawn by composer (double text).** Don't render text-heavy slides via `generate_image` — the model bakes text into the image AND composer adds drawtext on top. Use HTML/CSS screencap + `upload_asset` instead, and pass `update_slides({action:"set_text",newText:""})` to skip composer text for that slide.

## Cleanup

```bash
rm -rf /tmp/poppify-verify
```
