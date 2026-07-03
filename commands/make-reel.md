---
description: Generate a cinematic vertical reel via the Poppify MCP — photo-led or topic-led. Uses the poppify-build-reel skill for the canonical flow.
argument-hint: [photos or topic description]
---

The user wants to create a Poppify reel. Args provided: $ARGUMENTS

Invoke the `poppify-build-reel` skill and follow its flow:

1. If the user has no `apiKey` yet, call `register({ source: "claude" })` first. Surface the `signupBonusUrl` so they can claim 50 free seeds before paying.

2. Pick the entry based on what the user provided:
   - Photos (data URLs or http URLs) → `start_session_from_photos({ apiKey, photos, goal, audience?, platform? })`
   - Topic / brand intent (no photos) → `start_session({ apiKey, topic, audience, goal, platform })` then `refine_concept` to pick from the 5 returned concepts. **Topic-led sessions start with NO images** — every slide's `imageUrl` is empty. You must give at least one slide an image (search the library or `generate_image`, then `update_slides({action:"set_image", slideIndex, imageUrl})`) before `confirm`, or it refunds with `topic_led_no_images`. One image can serve every beat — `set_image` the same URL on each slide; captions + motion carry the variety.

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

4. Use `search_visual_library` and `get_music_library` BEFORE recommending `generate_image` / `generate_music` — library matches are free; generation is 5 seeds each. Attach images per-slide via `update_slides({action:"set_image", slideIndex, imageUrl})` (or `apply_session_patch({slides:[{index, imageUrl}]})`) — **not** the legacy pool. To swap music, `set_audio` / `apply_session_patch.audio`.

5. When ready: `get_price` to confirm seed cost, then `confirm({ sessionId, apiKey })`. Poll `get_result` every 20–30 seconds. When complete, hand the `videoUrl` to the user with the note that it expires in ~23h.

6. **Single-image reels**: when one image carries the whole reel, keep ONE `videoEffect` and leave `continuousEffect` on (default) so the camera makes one continuous move across the slides. Assigning a DIFFERENT effect per slide on a same-image run disables continuous smoothing and produces a visible reset at each cut.

7. **Optional Live Motion upgrade** (only AFTER the user has seen the cinematic baseline): to animate the subject inside a slide, `search_live_library` first (cache hits free), else `update_slides({action:"set_motion_mode", slideIndex, motionMode:"live", liveAction})` + `generate_live_motion` (10 seeds/clip, Veo 3.1 Lite). Recommend at most one live slide, usually the hook. Never apply it before the baseline render.

8. **Reel length is per-slide — no global knob.** Two DRIVERS: text length, or an explicit `update_slides({action:"set_duration", slideIndex, duration:N})` (2–15s, works on ANY slide, overrides the text-length formula). Two MEDIA FLOORS that always play in full: attached voiceover audio and a rendered live-motion (Veo) clip — so an **8s morph slide holds all 8s**; never pad a caption to keep a clip on screen. Lengthen a slide by writing more text OR set_duration.

For verification of the finished MP4, use `/poppify:verify-render`. For symptoms (wrong color, missing audio, etc.), use `/poppify:troubleshoot`.
