---
description: Download a rendered Poppify reel, run ffprobe + frame extract, surface a verdict on whether the render matches what was requested.
argument-hint: [videoUrl or sessionId]
---

The user wants to verify a Poppify render before shipping the URL. Args: $ARGUMENTS

**Capability gate first.** This is an OPTIONAL deep check that needs a local shell with `ffmpeg` + `ffprobe` + `curl`. If your client has no Bash/shell tool (e.g. Claude Desktop, web), STOP — you can't run it. Just give the user the `videoUrl` and, if they've reported a problem, use `/poppify:troubleshoot` (pure MCP-tool reasoning, no shell). Only proceed if `which ffprobe ffmpeg curl` succeeds.

Invoke the `poppify-render-debug` skill and follow it:

1. If args is a `sessionId`, call `get_result({ sessionId, apiKey })` to fetch the `videoUrl`. If args is already a URL, use it directly.

2. Download the MP4 to `/tmp/poppify-verify/render.mp4`.

3. Run `ffprobe -v error -print_format json -show_streams -show_format` and check:
   - Video stream present (h264, 720x1280, ~24fps)
   - Audio stream present (aac)
   - Duration matches the sum of per-slide durations (text-length-driven; voiceover/live clips play in full) within ±1s

4. Extract sample frames at 1fps and visually verify:
   - Caption text appears on intended slides
   - Caption color matches request
   - Caption position matches `textAnimation`
   - Image content matches the slide's `voiceoverShort`

5. Produce a Markdown verdict table. Green checks → hand the URL to the user (signed URL valid ~7 days). Red checks → surface the specific failure and recommend the corresponding `/poppify:troubleshoot` path.

6. Clean up `/tmp/poppify-verify/` when done.
