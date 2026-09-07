---
name: poppify-troubleshoot
description: Diagnose a Poppify reel that came out wrong. Use when the user says: "the reel came out wrong", "render failed", "no audio in the video", "no music", "captions are missing", "wrong color text", "text is green/blue/etc instead of white", "video URL doesn't open", "video URL expired", "seeds were charged but no video", "render is stuck", "render still loading after 5 minutes", "duration is wrong", "video too short / too long", "image has text baked in", "library search returned nothing", or any deviation from what they asked for. Walks the symptom → root cause → action decision tree.
---

# Poppify render troubleshooting

Find the symptom that matches, then take the suggested action. Most root causes here trace to schema drift, library-asset state, or text/duration mismatches — not random renderer failures.

## Symptom → Root cause → Action

### "The live-motion slide has no sound"

**Most likely**: the clip genuinely has no audio stream. Veo 3.1 always generates audio, but **Seedance and Kling are video-only** — check which provider rendered it (`set_model` / `get_capabilities.live_motion.availableModels`). A video-only provider falls back to the ElevenLabs path, so that slide is silent unless you attach narration.

**Also check**: in a same-image run, several slides share one clip at different offsets and the audio is taken **once**, from the slide that starts the run. Later slides in that run are silent by design.

**Not the cause any more**: the composer used to strip the track with `-an`. It no longer does — `VideoGenerator` extracts the clip's audio and mixes it through the voiceover path.

### "Two voices are talking over each other on one slide"

**Root cause**: a live clip's audio now REPLACES a generated voiceover on the same slide, so this shouldn't happen — if it does, the voiceover was attached to a *different* slide number than the live clip.

**Action**: `get_slide_plan` and confirm the `voiceoverSlides` index matches the slide carrying `motionMode:"live"`. Never budget `add_narration` for a live slide; its audio is already there.

### "No audio in the finished video"

**Most likely**: library audio asset was attached but `resolvedUrl` ended up empty (renderer can only download URLs, not asset IDs).

**Action**: `get_result` does NOT return a session object — it returns `status, videoUrl, signedVideoUrl, price, voiceover, slideCount, errorMessage`. For audio state call `get_slide_plan({sessionId})` and check the resolved soundtrack. If null/empty:
```
get_music_library({ apiKey })                      // get fresh assetIds
apply_session_patch({ sessionId, audio: { source: "library", assetId: <new id> } })
confirm({ sessionId, apiKey })                     // re-render (seeds re-charged)
```

If `resolvedUrl` is populated but audio still missing in the MP4: open `poppify-render-debug` and ffprobe the file — confirm the audio stream is actually absent, not just inaudible.

### "Caption text is the wrong color"

**Most likely**: `textColor` parameter was dropped because the deployed `apply_session_patch` schema doesn't expose it (schema/code drift bug).

**Action**: invoke the `poppify-schema-introspect` skill to verify. If `textColor` is in the schema but the render still ignored it, check the session via `get_slide_plan` (NOT `get_result`, which returns no session object). If the field is missing there, your `apply_session_patch` call dropped it client-side; re-call with the correct field name and re-`confirm`.

### "Captions don't appear on some slides"

**Two possibilities**:

1. **Intentional** — `update_slides({action:"set_text", slideIndex, newText:""})` was called on those slides (empty string = composer skips drawtext). Check the slide's text via `get_slide_plan({sessionId})` — if empty, this is the user's earlier patch landing.

2. **Photo-led copywriter trip** — `start_session_from_photos` ran but the inline copywriter failed silently and slides[] is empty. Falls back to concept hook + emotionalBeats (single-word labels). **Action**: call `refine_concept({ sessionId, conceptIndex, overrides: {...} })` to retry, then re-render.

### "Caption position is wrong"

**Most likely**: the agent set `textAnimation` and `textPosition` to conflicting values (typewriter wants top, you forced bottom). The renderer respects the explicit `textPosition`, but the animation style may render awkwardly at the forced position.

**Action**: pick the right animation for the position (renderer anchors):
- upper-third text (centered ~33% height) → `textAnimation: "typewriter"`
- middle text (vertically centered) → `textAnimation: "bold_captions"`
- lower-third text (block bottom ~84%) → `textAnimation: "phrase_reveal"`

Don't override `textPosition` unless you know why. The default position comes from the animation style and usually looks best.

### "Render failed / seeds charged but no video"

**Action**:
```
get_result({ sessionId, apiKey })
```

If `status === "failed"`: the render error is in `errorMessage` (get_result returns no session object). Confirm seeds were refunded (`wallet({action:"balance"})` should show the seeds back) — Poppify refunds on render-error automatically. If the balance is not refunded, file via `submit_feedback`.

If `status === "rendering"` for > 5 minutes: the render is genuinely stuck (rare). File via `submit_feedback({ apiKey, sessionId, frictionPoints: ["render stuck in status=rendering for >5 minutes"] })`.

### "I got the video URL but it doesn't open"

**Most likely**: the signed GCS URL expired. TTL is ~7 days (`get_result` returns `videoUrlExpiresAtIso` with the exact time).

**Action**:
```
get_result({ sessionId, apiKey })
```

If `videoUrl` returned but the URL doesn't open: the URL has expired and Poppify doesn't persist videos beyond the ~7-day TTL (cost-control). Re-`confirm` charges seeds again. Tell the user to save the file within the window next time (shell clients: `curl` a local copy right after render).

### "Video duration is wrong"

**Most likely**: slide length is driven by text length OR an explicit per-slide duration. A reel shorter than expected means the slides have too little text and no explicit duration.

**Action**: two levers — write more text on the short slides (`update_slides({action:"set_text", newText:"longer text"})`), OR set an explicit hold with `update_slides({action:"set_duration", slideIndex, duration:N})` (2–15s, works on ANY slide, overrides the text-length formula). There's no session-level duration knob. Note: **voiceover audio and rendered live-motion clips always play in full** — they're media floors, never scaled down.

### "The live-motion / morph clip is cut short" (e.g. an 8s Veo clip only plays ~2s)

**Most likely**: on older builds the live slide followed the caption's text-length instead of the clip length, truncating the Veo clip. The duration resolver now treats a rendered live clip as a **media floor** (always plays in full), so this is fixed on current builds.

**Action**: if a live clip is still cut short, set an explicit floor with `update_slides({action:"set_duration", slideIndex, duration:8})` on that live slide, then re-render. Do NOT try to fix it by padding the caption — the clip length binds on its own now.

### "The image in slide N looks wrong / has text baked in"

**Most likely**: `add_slide_image` was called for a text-heavy concept and Gemini Imagen garbled the literal text into the image, AND composer drew drawtext on top — so the slide has double text.

**Action**:
- For text-heavy slides (terminal frames, install commands, stat callouts, end cards): render text-on-bg locally via HTML/CSS screencap + `upload_asset`, then attach via `update_slides({action:"set_image", slideIndex, imageUrl:accessUrl})` + `update_slides({action:"set_text", slideIndex, newText:""})` (so composer doesn't add text on top). (`set_image` is the way to attach a slide's image; insert/splice pool edits go through `apply_session_patch({visualEdits:[...]})`.)
- For non-text-heavy slides where AI image is fine: just re-generate with a sharper prompt — workshop it with `suggest_prompt({kind:"image"})` first (free).

### "Library search returned nothing relevant"

**Action**: refine the query. Library scoring weights **Visual Hint Match (tag/keyword)** at 50/100 points — generic queries score low. Use specific keywords from the slide's `voiceoverShort` or concept hook.

If still nothing > score 40: fall through to `add_slide_image` (5 seeds). Workshop the prompt with `suggest_prompt({kind:"image"})` for free before generating.

### "Single-image reel has jerky / resetting motion at each cut"

**Most likely**: different `videoEffect` values were set per slide on a same-image run, which disables continuous-curve smoothing. Each slide then gets its own independent camera move, so the motion visibly resets at every cut instead of flowing as one continuous shot.

**Action**: for a reel where one image carries all slides, use ONE session-wide `videoEffect` (e.g. `push_in`) and set `continuousEffect: true` (default `false` — you must set it) — the renderer then makes a single continuous camera move across the whole reel via a global frame offset. Variety comes from the changing captions, not from per-slide motion. Only assign per-slide effects when slides have *different* images.

### "confirm() says 'needs at least one image attached' but I set images"

**Most likely**: images were placed per-slide via `set_image` but you're on an older deployment whose confirm gate only checked the legacy pool. On current builds, `confirm()` accepts any session where a slide carries its own `imageUrl`.

**Action**: confirm at least one slide has an image populated (via `get_slide_plan`). If it is and confirm still rejects, the deployment is stale — `apply_session_patch({visualEdits:[{action:"insert_before", slideIndex:0, source:"user_url", url}]})` seeds the pool as a fallback, or file `submit_feedback`. Do NOT use a pool `replace` edit on a topic-led session — the pool is empty so it errors; `update_slides({action:"set_image", slideIndex, imageUrl})` is the way to replace a slide's image.

### "Live motion didn't apply / subject isn't moving"

**Most likely**: `animate_slide` was never called, or the slide's `motionMode` wasn't set to `live` first. Live motion is a deliberate 2-step upgrade, not a default.

**Action**: `update_slides({action:"set_motion_mode", slideIndex, motionMode:"live", liveAction:"<verb>"})` THEN `animate_slide({sessionId, slideIndex})` (10 seeds, or free on a `search_live_library` cache hit ≥ 60). Confirm the cinematic baseline was rendered and reviewed first — live motion is offered only after that.

## When to file feedback

If the symptom doesn't match any of the above, OR you've followed the action and the issue persists, call:

```
submit_feedback({
  apiKey,
  sessionId,                          // optional but include when you have one
  frictionPoints: ["<the symptom + what you tried, specific>"],
  suggestedImprovements: ["<concrete tool/doc/flow change>"],   // optional
  notes: "..."                        // optional free-text context
})
```

(There is no `kind`/`description` field — describe bugs in `frictionPoints`, improvement ideas in `suggestedImprovements`.)

Don't re-file known-resolved issues — check the "Recently addressed" block in the MCP server instructions before filing.
