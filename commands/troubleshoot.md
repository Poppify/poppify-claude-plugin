---
description: Diagnose a Poppify reel that came out wrong. Walks the symptom → root cause → action decision tree.
argument-hint: [symptom or sessionId]
---

The user is reporting a problem with a Poppify render. Args: $ARGUMENTS

Invoke the `poppify-troubleshoot` skill and walk the user through:

1. **Identify the symptom** — match the user's description to one of these categories:
   - No audio in finished video
   - Caption text wrong color
   - Captions missing on some slides
   - Caption position wrong
   - Render failed / seeds charged but no video
   - Video URL doesn't open
   - Duration wrong
   - Image looks wrong (or has text baked in)
   - Library search returned nothing

2. **Apply the corresponding action** from the skill's decision tree. Most fixes are one tool call + re-`confirm`.

3. **Verify the fix** — if your client has a shell with ffmpeg, `/poppify:verify-render` gives a deep check; otherwise just re-`get_result` and confirm the reported symptom is gone (e.g. audio now attached, text finalized). Verification is optional, not a gate.

4. **If the symptom doesn't match any category**, file via `submit_feedback({ apiKey, sessionId, kind: "bug", description: ... })`.

Do NOT re-flag issues already listed in the MCP server's "Recently addressed" block — they're known-resolved.
