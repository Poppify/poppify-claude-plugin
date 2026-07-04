---
name: poppify-build-reel
description: Canonical photo-led / topic-led flow for the Poppify MCP. Use whenever the user wants: a reel, short video, vertical video, Instagram reel, TikTok video, TikTok, YouTube Short, YouTube Shorts, Facebook reel, FB reel, 15-second video, 30-second video, 60-second video, photo slideshow, photo to video, animate photos, photo animation, slideshow video, social media video, content for social, brand video, product reel, ad creative for social, before/after reel, transformation video, story reel, hook video, or any vertical short-form video for IG/TikTok/YT/FB. Covers the free customize loop and the single paid confirm step. Base render = 1 seed (~$0.06). 50 free seeds on signup. Use poppify-troubleshoot if the render comes out wrong; poppify-render-debug to verify the finished MP4.
---

# Building a Poppify reel — the canonical flow

Poppify is FREE for everything except `confirm` (1 seed base render) and the `generate_*` tools (5 seeds each for AI image / music / voiceover; `generate_live_motion` is 10 seeds per clip). You can iterate the configuration **infinitely** before render and not spend anything.

> **Think in SLIDES, not a photo pool.** Every beat owns its image at `slides[i].imageUrl`. Attach or swap a beat's visual with `update_slides({action:"set_image", slideIndex, imageUrl})`, or set several at once via `apply_session_patch({slides:[{index, imageUrl}]})`. The legacy `update_visual` operates on a `photoUrls` pool and **fails when the pool is empty** (every topic-led session) — prefer `set_image`.

## Step 1 — Mint a wallet (one-time, free)

If the user hasn't registered yet:

```
register({ source: "claude" })
```

Returns `apiKey` and `signupBonusUrl`. **Surface the signupBonusUrl** — opening it and signing in with Google grants 50 free seeds (≈ one fully-loaded reel WITH AI image + AI music + AI voiceover). Don't ask the user to pay before they've claimed this.

Store the `apiKey` — every subsequent call needs it.

## Step 2 — Start a session

**Photo-led** (user has photos already):
```
start_session_from_photos({
  apiKey,
  photos: [...],            // data: URLs or http(s) URLs, 1–10
  goal: "educate" | "sell" | "connect" | "prove" | "entertain",
  audience: "...",          // optional but biases narrative meaningfully
  theme: "...",             // optional free-text intent
  platform: "instagram" | "tiktok" | "youtube_shorts" | "facebook"
})
```

Returns: slides[] (each with `voiceoverShort` text), caption, hashtags, callToAction, picked recipe.

**Topic-led** (no photos, brand or topic input):
```
start_session({ apiKey, topic, audience, goal, platform })
```

Returns 5 concept options. Pick one with `refine_concept`. **A topic-led session starts with NO images** — every slide's `imageUrl` is empty. You MUST give at least one slide an image (Step 4) before `confirm`, or it refunds with `topic_led_no_images`. The `recipeDistribution` field shows which recipes the 5 concepts span; if they collapsed off-brief (e.g. you wanted a tribute/hype angle but got all confessional), call `start_session` again with an explicit `recipe` parameter (`list_recipes()` for IDs).

### Decide FIRST: one image, or one per beat?

Before generating anything, ask whether **one image can carry the whole reel**. For a single-subject / hero / cinematic reel (recipe `visualType: "hero_image"`), it usually can — the camera motion and the changing captions ARE the variety; the image stays the star. Generating four images for four beats is the wrong default — it costs 4× the seeds AND drifts the subject's identity across slides.

- **One image across all beats** (recommended for single-subject reels): `generate_image` once, then `set_image` the SAME URL on every slide. Keep ONE `videoEffect` + `continuousEffect` on so the camera makes one continuous move (see Gotchas). 5 seeds total for imagery.
- **One image per beat** (only when beats are genuinely distinct scenes — before/after, multi-step, comparison): generate/search per slide. N × 5 seeds.

## Step 3 — Refine (FREE, unlimited iteration)

The single most efficient call here is **`apply_session_patch`** — it batches per-slide text + music + visual edits + production knobs + textColor + per-slide motion into ONE round-trip:

```
apply_session_patch({
  sessionId,
  // Per-slide caption text (empty string = no caption on that slide)
  slides: [
    { index: 0, text: "Your literal slide 1 caption" },
    { index: 1, text: "" },               // skip caption on this slide
    { index: 2, text: "..." }
  ],
  // Production knobs
  textColor: "#FFFFFF",                    // hex; overrides recipe palette
  textPosition: "bottom",                  // top | center | bottom
  textAnimation: "phrase_reveal",          // typewriter | bold_captions | phrase_reveal
  videoEffect: "ken_burns",                // session-wide motion
  audioMood: "uplifting",
  // NOTE: no SESSION-level `duration` knob. Per-slide length is resolved as:
  // media floors (voiceover audio + a rendered live-motion clip) ALWAYS play in
  // full; then explicit set_duration (any slide, capped 2–15s) or text-length
  // drive the rest; 4s default. To change runtime: write more/fewer words, use
  // set_duration on any slide, or add/remove slides.
  // Per-slide motion overrides (takes precedence over session-wide)
  slideEffects: [
    { slideIndex: 3, videoEffect: "focus_pull" }  // end card → minimal motion
  ],
  // Music attach (free — library audio is resolved to a playable URL here)
  audio: { source: "library", assetId: "..." }
})
```

If you'd rather iterate one knob at a time, the per-action tools all work:
- `update_slides({ action: "set_text", slideIndex, newText })` — literal text per slide
- `update_slides({ action: "set_image", slideIndex, imageUrl })` — set/swap a beat's image (slide-first; this is the way)
- `customize(...)` — production knobs
- `set_audio(...)` — attach music
- `update_visual(...)` — legacy pool replace/insert (works only on photo-led pool sessions; use `set_image` instead)

**Search the library FIRST** before generating:
- `search_visual_library({ apiKey, query, limit })` for images — score ≥ 40 should beat AI gen
- `get_music_library({ apiKey, mood, genre })` for music

## Step 4 — Generate AI assets only when needed

These cost 5 seeds each. Always workshop the prompt for free first:

```
suggest_image_prompt({ apiKey, slideContext })   // FREE prompt refinement
generate_image({ apiKey, prompt, style })        // 5 seeds, returns image URL
// Then attach via update_slides({action:"set_image", slideIndex, imageUrl})
//   — or apply_session_patch({slides:[{index, imageUrl}, ...]}) for several beats.
//   For a single-image reel, set_image the SAME URL on every slide.

suggest_music_prompt({ apiKey, mood, genre })    // FREE prompt refinement
generate_music({ apiKey, prompt, durationSeconds }) // 5 seeds, returns URL (max 30s)
// Then attach via apply_session_patch.audio or set_audio

list_voices({ apiKey })                          // FREE voice catalog
generate_voiceover({ apiKey, scripts, voiceId }) // 5 seeds per batch
// Then attach via apply_session_patch.voiceoverSlides or set_voiceover
```

## Step 5 — Verify, then confirm

Use `get_price({ sessionId })` to see the seed total. Then:

```
confirm({ sessionId, apiKey, buyerEmail })
```

Charges seeds. Render typically completes in 30–120s. Poll:

```
get_result({ sessionId, apiKey })
```

When `status === "complete"`, you get a `videoUrl` (signed GCS URL, expires in ~23h). **Download it immediately** — the URL is short-lived. Don't tell the user "we emailed it" — Poppify does not email rendered reels; the URL is the only delivery surface.

## Step 6 — OPTIONAL: Live Motion (a per-slide upgrade, only AFTER baseline review)

Live Motion animates the **subject inside a still** (blink, breath, micro-gesture, a small action) using Veo 3.1 Lite image-to-video, while the FFmpeg camera motion layers on top. It is an OPTIONAL upgrade, NOT a default — the flow is **render the cinematic baseline → user reviews → optionally upgrade selected slides to live**. Never apply it before the user has seen the cinematic render; you'd risk burning seeds on a reel they might already love.

> **For before/after transformations (`endFrameUri` first/last-frame interpolation), invoke the `poppify-live-motion` skill first** — bridged renders need a composition-locked end frame and an authored journey `overridePrompt`; the auto prompt is wrong for that mode.

When the user wants it:
1. `search_live_library({ apiKey, imageHash?, actionKeywords, durationSeconds, provider:"veo-3.1-lite" })` FIRST — cache hits (score ≥ 60) attach for **zero seeds**.
2. `suggest_live_action({ sessionId, slideIndex })` → pick a motion verb; `preview_live_prompt` is free.
3. `update_slides({ action:"set_motion_mode", slideIndex, motionMode:"live", liveAction:"<verb>", liveDurationSeconds:<4|6|8> })`.
4. `generate_live_motion({ sessionId, slideIndex })` — **10 seeds** per clip (capped 8s), or free on a cache hit. ~30–90s wall-clock.
5. `confirm` again to re-render with the live slide.

Recommend **at most one** live slide, usually the hook (slide 0) — a living subject in the first frame stops the scroll. Quote the planner's per-slide score to justify the pick.

## Gotchas worth knowing

- **Don't burn seeds on text-heavy slides.** Gemini Imagen garbles literal text (terminal frames, install commands, stat callouts). For those: HTML/CSS screencap locally → `upload_asset` → `update_slides({action:"set_image", slideIndex, imageUrl:accessUrl})`. ZERO seeds.
- **Single-image reel → ONE continuous camera move.** When one image carries all slides, keep a SINGLE `videoEffect` and leave `continuousEffect` on (default `true`). The renderer then makes one continuous move across the whole reel via a global frame offset. **Assigning a DIFFERENT effect per slide on a same-image run disables continuous smoothing** — each slide gets its own independent move and the motion visibly resets at every cut. Only use per-slide `slideEffects` when the slides have *different* images.
- **Composer draws caption text BY DEFAULT** from `slides[i].voiceoverShort`. Empty string = skipped.
- **Slide duration: two DRIVERS + two MEDIA FLOORS, no global control.** Media floors always play in full: (1) attached **voiceover** audio, (2) a rendered **live-motion (Veo) clip** — so an 8s morph slide holds all 8s even under a short caption (you do NOT pad the caption). Drivers: (3) **EXPLICIT** `update_slides({action:"set_duration", slideIndex, duration:N})` on ANY slide — overrides text-length, capped 2–15s, never shortens the media floor; (4) **TEXT-LENGTH** `(words/2.6)*1.2` otherwise; 4s default for a blank slide. Lengthen a slide: write MORE text OR use set_duration. `customize({duration})` is rejected — no session-level control. **Use text** for normal captioned slides; **use set_duration** for text-baked cards or any precise hold.
- **Voiceover auto-detaches when text changes.** The text you write IS the voiceover script. `update_slides({action:"set_text"})` on a slide with attached voiceover detaches it (old audio of wrong words). Call `generate_voiceover` ONLY after text is finalized — otherwise you waste 5 seeds per text edit.
- **Text cards (terminal screencaps, end cards)**: `set_image` an image with text already baked in, then `update_slides({action:"set_text", slideIndex, newText:""})`. Empty string suppresses composer text overlay. Hold it as long as you want with `update_slides({action:"set_duration", slideIndex, duration:N})` (2–15s) — set_duration works on any slide, so you no longer have to blank the caption first just to control the hold. Blank slide with no explicit duration falls back to 4s.
- **Sceneboard recipes auto-lock motion across all panels** — per-slide variation is ignored when visualType is sceneboard.
- **Library audio resolves URLs at attach time** — if `set_audio({source:"library"})` errors with "asset not found", pick a different `assetId` from `get_music_library`.

## When something looks wrong with the finished video

Invoke the `poppify-troubleshoot` skill. It has the decision tree for common symptoms (wrong colors, missing audio, caption drops, duration mismatch).
