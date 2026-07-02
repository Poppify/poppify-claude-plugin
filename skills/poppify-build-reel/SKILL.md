---
name: poppify-build-reel
description: Canonical photo-led / topic-led flow for the Poppify MCP. Use whenever the user wants: a reel, short video, vertical video, Instagram reel, TikTok video, TikTok, YouTube Short, YouTube Shorts, Facebook reel, FB reel, 15-second video, 30-second video, 60-second video, photo slideshow, photo to video, animate photos, photo animation, slideshow video, social media video, content for social, brand video, product reel, ad creative for social, before/after reel, transformation video, story reel, hook video, or any vertical short-form video for IG/TikTok/YT/FB. Covers the free customize loop and the single paid confirm step. Base render = 1 seed (~$0.06). 50 free seeds on signup. Use poppify-troubleshoot if the render comes out wrong; poppify-render-debug is an OPTIONAL deep MP4 check for clients that have a shell with ffmpeg (e.g. Claude Code) — skip it on shell-less clients.
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
  // NOTE: no `duration` knob. Reel duration is driven by per-slide content
  // (voiceover length > text length > explicit slide.duration > 4s default).
  // To change runtime: write more/fewer words, add/remove slides, or set
  // slide.duration on blank text-baked cards.
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

When `status === "complete"`, you get a `videoUrl` (signed GCS URL, valid ~7 days). **Give the user the URL right away** — it's the only delivery surface (Poppify does not email rendered reels). If your client has a shell (e.g. Claude Code), it's good practice to also save a local copy (`curl -L -o reel.mp4 "<videoUrl>"`) so the user keeps it past the expiry — but that's an enhancement, not a requirement. Shell-less clients (Claude Desktop, web) simply share the URL and tell the user to download it from their browser.

## Step 6 — OPTIONAL: Live Motion (a per-slide upgrade, only AFTER baseline review)

Live Motion animates the **subject inside a still** (blink, breath, micro-gesture, a small action) using Veo 3.1 Lite image-to-video, while the FFmpeg camera motion layers on top. It is an OPTIONAL upgrade, NOT a default — the flow is **render the cinematic baseline → user reviews → optionally upgrade selected slides to live**. Never apply it before the user has seen the cinematic render; you'd risk burning seeds on a reel they might already love.

When the user wants it:
1. `search_live_library({ apiKey, imageHash?, actionKeywords, durationSeconds, provider:"veo-3.1-lite" })` FIRST — cache hits (score ≥ 60) attach for **zero seeds**.
2. `suggest_live_action({ sessionId, slideIndex })` → pick a motion verb; `preview_live_prompt` is free.
3. `update_slides({ action:"set_motion_mode", slideIndex, motionMode:"live", liveAction:"<verb>", liveDurationSeconds:<4|6|8> })`.
4. `generate_live_motion({ sessionId, slideIndex })` — **10 seeds** per clip (capped 8s), or free on a cache hit. ~30–90s wall-clock.
5. `confirm` again to re-render with the live slide.

Recommend **at most one** live slide, usually the hook (slide 0) — a living subject in the first frame stops the scroll. Quote the planner's per-slide score to justify the pick.

## Step 7 — OPTIONAL: consistent frames (`generate_frames`) + before/after interpolation (`endFrameUri`)

Two optional capabilities that unlock character-consistent reels and before/after transformations. They compose but are used at **different moments** — don't conflate them.

**`generate_frames` — a new still of an EXISTING subject (image stage, before motion).** Same cost as `generate_image` (5 seeds). Give it a reference subject (`referenceAssetId` or `referenceImageUrl` — a prior `generate_image`/`generate_frames` result, or an uploaded photo) plus a prompt, and it returns ONE frame of that same subject in the new pose/expression/scene you describe. **Each frame is generated with intent — one deliberate call per frame, never a random batch.** Two consumers:
- **Same subject across slides** → `generate_frames` for each slide, then `update_slides({action:"set_image", slideIndex, imageUrl})`, animate normally. This is the consistency path — **no last frame**.
- **A specific "after" state** (clean room, styled product, logo revealed) → becomes the end frame below.

**`endFrameUri` — send a last frame so ONE clip travels start → end. Opt-in, transitions only.** Set it on `set_motion_mode`; Veo bridges the slide's start image → the end frame instead of free-running. Use ONLY for before/after within a slide, a controlled morph, or a deterministic landing. **Do NOT** send a last frame for ordinary micro-motion (breath/blink/gaze) — that's plain i2v and wants no endpoint.

Before/after transformation slide (the chain):
```
generate_image (or photo)                                    # the "before" (slide start image)
generate_frames({ referenceImageUrl:<before>, prompt:"same subject, <after state>" })   # the "after"
update_slides({ action:"set_motion_mode", slideIndex, motionMode:"live", liveAction:"<verb>", endFrameUri:<after url> })
preview_live_prompt → generate_live_motion                    # Veo interpolates before → after
```

Decision shortcut: **same subject across slides → `generate_frames` then `set_image`, no last frame. Before/after or morph within one slide → `generate_frames` for the "after", then send it as `endFrameUri`.**

> ⚠️ Start+end interpolation on Veo 3.1 Lite (the default provider) is newly wired — confirm the render actually bridged the two frames before promising a client it always will; if a run ignores the end frame, fall back to two slides with a hard cut.

## Gotchas worth knowing

- **Don't burn seeds on text-heavy slides.** Gemini Imagen garbles literal text (terminal frames, install commands, stat callouts). On a shell-capable client (Claude Code): invoke the `poppify-text-card` skill to render a pixel-perfect card locally (cross-platform) → `upload_asset` → `set_image`. ZERO seeds. On a shell-less client (Claude Desktop/web): let the composer draw the text as a caption (`set_text` — crisp drawtext, not garbled), and reserve rendered cards for specialty typography you can ingest via `upload_asset({sourceUrl|dataBase64})`.
- **Single-image reel → ONE continuous camera move.** When one image carries all slides, keep a SINGLE `videoEffect` and leave `continuousEffect` on (default `true`). The renderer then makes one continuous move across the whole reel via a global frame offset. **Assigning a DIFFERENT effect per slide on a same-image run disables continuous smoothing** — each slide gets its own independent move and the motion visibly resets at every cut. Only use per-slide `slideEffects` when the slides have *different* images.
- **Composer draws caption text BY DEFAULT** from `slides[i].voiceoverShort`. Empty string = skipped.
- **Slide duration is a hierarchy, no global control.** Four rules, top wins: (1) voiceover audio length when attached (binding); (2) text-length formula `(words/2.6)*1.2` when text exists; (3) explicit `slide.duration` for blank text-baked cards; (4) 4s default for blank slides. To make a slide last longer: write MORE text. `customize({duration})` is rejected — no session-level duration control.
- **Voiceover auto-detaches when text changes.** The text you write IS the voiceover script. `update_slides({action:"set_text"})` on a slide with attached voiceover detaches it (old audio of wrong words). Call `generate_voiceover` ONLY after text is finalized — otherwise you waste 5 seeds per text edit.
- **Text cards (terminal screencaps, end cards)**: `set_image` an image with text already baked in, then `update_slides({action:"set_text", slideIndex, newText:""})`. Empty string suppresses composer text overlay. Slide duration falls back to 4s.
- **Sceneboard recipes auto-lock motion across all panels** — per-slide variation is ignored when visualType is sceneboard.
- **Library audio resolves URLs at attach time** — if `set_audio({source:"library"})` errors with "asset not found", pick a different `assetId` from `get_music_library`.

## When something looks wrong with the finished video

Invoke the `poppify-troubleshoot` skill. It has the decision tree for common symptoms (wrong colors, missing audio, caption drops, duration mismatch).
