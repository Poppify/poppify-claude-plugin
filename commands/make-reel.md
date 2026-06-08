---
description: Generate a cinematic vertical reel via the Poppify MCP — photo-led or topic-led. Uses the poppify-build-reel skill for the canonical flow.
argument-hint: [photos or topic description]
---

The user wants to create a Poppify reel. Args provided: $ARGUMENTS

Invoke the `poppify-build-reel` skill and follow its flow:

1. If the user has no `apiKey` yet, call `register({ source: "claude" })` first. Surface the `signupBonusUrl` so they can claim 50 free seeds before paying.

2. Pick the entry based on what the user provided:
   - Photos (data URLs or http URLs) → `start_session_from_photos({ apiKey, photos, goal, audience?, platform? })`
   - Topic / brand intent (no photos) → `start_session({ apiKey, topic, audience, goal, platform })` then `refine_concept` to pick from the 5 returned concepts

3. Review the returned slides + caption + hashtags + CTA with the user. Default to ONE bulk call to apply any edits:
   ```
   apply_session_patch({
     sessionId,
     slides: [{index: 0, text: "..."}, ...],
     textColor: "#FFFFFF",         // or any hex
     textAnimation: "phrase_reveal",
     videoEffect: "ken_burns",
     audioMood: "uplifting",
     audio: { source: "library", assetId: "..." }
   })
   ```

4. Use `search_visual_library` and `get_music_library` BEFORE recommending `generate_image` / `generate_music` — library matches are free; generation is 10 seeds each.

5. When ready: `get_price` to confirm seed cost, then `confirm({ sessionId, apiKey })`. Poll `get_result` every 20–30 seconds. When complete, hand the `videoUrl` to the user with the note that it expires in ~23h.

For verification of the finished MP4, use `/poppify:verify-render`. For symptoms (wrong color, missing audio, etc.), use `/poppify:troubleshoot`.
