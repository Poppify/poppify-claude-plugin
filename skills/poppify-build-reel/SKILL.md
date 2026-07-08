---
name: poppify-build-reel
description: Canonical photo-led / topic-led flow for the Poppify MCP. Use whenever the user wants: a reel, short video, vertical video, Instagram reel, TikTok video, TikTok, YouTube Short, YouTube Shorts, Facebook reel, FB reel, 15-second video, 30-second video, 60-second video, photo slideshow, photo to video, animate photos, photo animation, slideshow video, social media video, content for social, brand video, product reel, ad creative for social, before/after reel, transformation video, story reel, hook video, or any vertical short-form video for IG/TikTok/YT/FB. Covers the free customization loop and the single paid confirm step. Base render = 1 seed (~$0.06). 50 free seeds on signup. Use poppify-troubleshoot if the render comes out wrong; poppify-render-debug to verify the finished MP4.
---

# Building a Poppify reel — the canonical flow

Poppify is FREE for everything except `confirm` (1 seed base render) and the `generate_*` tools (5 seeds each for AI image / music / voiceover; `animate_slide` is 10 seeds per clip). You can iterate the configuration **infinitely** before render and not spend anything.

> **Think in SLIDES, not a photo pool.** Every beat owns its image at `slides[i].imageUrl`. Attach or swap a beat's visual with `update_slides({action:"set_image", slideIndex, imageUrl})`, or set several at once via `apply_session_patch({slides:[{index, imageUrl}]})`. For insert/splice edits to the slide sequence, use `apply_session_patch({visualEdits:[...]})` — `set_image` is the way to set or swap an existing beat's image.

## Step 1 — Mint a wallet (one-time, free)

If the user hasn't registered yet:

```
register()                       // optional: { label: "claude" } for provenance
```

Returns `apiKey` and `signupBonusUrl`. **Surface the signupBonusUrl** — opening it and signing in with Google grants 50 free seeds (≈ 50 base renders, or ~3 fully-loaded reels WITH AI image + AI music + AI voiceover at ~16 seeds each). Don't ask the user to pay before they've claimed this.

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
  platform: "instagram" | "tiktok" | "youtube_shorts" | "facebook",
  aspectRatio: "9:16" | "16:9"   // OPTIONAL — defaults to 9:16 (vertical). See below.
})
```

Returns: slides[] (each with `voiceoverShort` text), caption, hashtags, callToAction, picked recipe.

**Topic-led** (no photos, brand or topic input):
```
start_session_from_topic({ apiKey, topic, audience, goal, platform, aspectRatio })
```

**Aspect ratio is a SESSION property — set it ONCE here.** It defaults to `9:16`
(vertical) and everything downstream inherits it: `add_slide_image` / `generate_frames`
stills, live-motion clips, library search/match, and the render canvas. Only pass
`aspectRatio: "16:9"` when the user explicitly wants a **landscape** video (e.g. a
regular 16:9 YouTube upload — NOT a Short). Don't mix orientations: a 16:9 still in a
9:16 session (or vice versa) gets cropped/letterboxed, and `add_slide_image` will warn.

Returns 5 concept options. Pick one with `refine_concept`. **A topic-led session starts with NO images** — every slide's `imageUrl` is empty. You MUST give at least one slide an image (Step 4) before `confirm`, or it refunds with `topic_led_no_images`. The `recipeDistribution` field shows which recipes the 5 concepts span; if they collapsed off-brief (e.g. you wanted a tribute/hype angle but got all confessional), call `start_session_from_topic` again with an explicit `recipe` parameter (`recipes()` for IDs).

### Decide FIRST: one image, or one per beat?

Before generating anything, ask whether **one image can carry the whole reel**. For a single-subject / hero / cinematic reel (recipe `visualType: "hero_image"`), it usually can — the camera motion and the changing captions ARE the variety; the image stays the star. Generating four images for four beats is the wrong default — it costs 4× the seeds AND drifts the subject's identity across slides.

- **One image across all beats** (recommended for single-subject reels): `add_slide_image` once, then `set_image` the SAME URL on every slide. Keep ONE `videoEffect` + `continuousEffect` on so the camera makes one continuous move (see Gotchas). 5 seeds total for imagery.
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
  videoEffect: "push_in",                  // session-wide motion (canonical: push_in, pull_out, lateral_pan, vertical_pan, focus_pull, epic_parallax, static)
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
- `apply_session_patch({...})` — production knobs (one knob per call works fine)
- `apply_session_patch({audio:{source, assetId}})` — attach music
- `apply_session_patch({visualEdits:[...]})` — insert/splice slides (`set_image` is the way to set a beat's image)

**Search the library FIRST** before generating:
- `search_visual_library({ apiKey, keywords, limit })` for images (keywords = string or array: subject + mood + scene) — score ≥ 40 should beat AI gen
- `get_music_library({ apiKey, mood, genre })` for music

## Step 4 — Generate AI assets only when needed

These cost 5 seeds each. Always workshop the prompt for free first:

```
suggest_prompt({ apiKey, kind: "image", sessionId, slideIndex })  // FREE — pass sessionId+slideIndex
                                                 //   for the slide's composer plan (or subjectDescription when no session)
add_slide_image({ apiKey, prompt, visualStyle, sessionId })  // 5 seeds, returns image URL
//   Pass sessionId so the still inherits the session's aspect ratio (vertical by
//   default). The render canvas is fixed to the session aspect, so a still that
//   doesn't match it is cropped — passing sessionId keeps them aligned.
// Then attach via update_slides({action:"set_image", slideIndex, imageUrl})
//   — or apply_session_patch({slides:[{index, imageUrl}, ...]}) for several beats.
//   For a single-image reel, set_image the SAME URL on every slide.

suggest_prompt({ apiKey, kind: "music", userInput })     // FREE — userInput = natural-language music description
add_soundtrack({ apiKey, prompt, durationSeconds }) // 5 seeds, returns URL (default 30s, max 300s)
// Then attach via apply_session_patch({audio:{source:"user_url", url}})

list_voices({ apiKey })                          // FREE voice catalog
add_narration({ apiKey, scripts, voiceId }) // 5 seeds per batch
// Then attach via apply_session_patch({voiceoverSlides})
```

## Step 5 — Verify, then confirm

Use `get_result({ sessionId })` to see the seed total — pre-confirm it returns the exact price breakdown. Then:

```
confirm({ sessionId, apiKey, buyerEmail })
```

Charges seeds. Render typically completes in 30–120s. Poll:

```
get_result({ sessionId, apiKey })
```

When `status === "complete"`, you get a `videoUrl` (signed GCS URL, valid ~7 days — `videoUrlExpiresAtIso` in the response has the exact expiry). Hand it to the user right away and tell them to save it before it expires; on shell clients also `curl` a local copy as the durable backup. Don't tell the user "we emailed it" — Poppify does not email rendered reels; the URL is the only delivery surface.

## Step 6 — OPTIONAL: Live Motion (a per-slide upgrade, only AFTER baseline review)

Live Motion animates the **subject inside a still** (blink, breath, micro-gesture, a small action) using Veo 3.1 Lite image-to-video, while the FFmpeg camera motion layers on top. It is an OPTIONAL upgrade, NOT a default — the flow is **render the cinematic baseline → user reviews → optionally upgrade selected slides to live**. Never apply it before the user has seen the cinematic render; you'd risk burning seeds on a reel they might already love.

> **For before/after transformations (`endFrameUri` first/last-frame interpolation), invoke the `poppify-live-motion` skill first** — bridged renders need a composition-locked end frame and an authored journey `overridePrompt`; the auto prompt is wrong for that mode.

When the user wants it (same order as the server instructions' lifecycle):
1. `suggest_live_action({ sessionId, slideIndex })` (FREE) → pick a motion verb with the user.
2. `animate_slide({ sessionId, slideIndex, dryRun:true, liveAction })` (FREE) — preview the exact Veo prompt; iterate until it reads right.
3. `search_live_library({ apiKey, imageHash?, actionKeywords:<finalized action>, durationSeconds, provider:"veo-3.1-lite" })` — cache hits (score ≥ 60) attach for **zero seeds**.
4. `update_slides({ action:"set_motion_mode", slideIndex, motionMode:"live", liveAction:"<verb>", liveDurationSeconds:<4|6|8> })`.
5. `animate_slide({ sessionId, slideIndex })` — **10 seeds** per clip (capped 8s), or free on a cache hit. ~30–90s wall-clock.
6. `confirm` again to re-render with the live slide.

Recommend **at most one** live slide, usually the hook (slide 0) — a living subject in the first frame stops the scroll. Quote the planner's per-slide score to justify the pick.

## Step 7 — OPTIONAL: publish or schedule (linked accounts only)

Rendering and publishing are separate. `confirm` produces the MP4 and files a `ready` post in the user's Poppify portfolio; `publish_post` is what sends it to social channels. Both `publish_post` and `portfolio` are FREE.

Preconditions, in order:
1. **Wallet linked** to the user's Poppify app account: `wallet({action:"link"})` (QR / code). Channels live on the app account.
2. **Channels connected** — Instagram / TikTok / YouTube / Facebook OAuth happens in the Poppify mobile app only. `portfolio({action:"list"})` shows portfolios + connected channels; multi-brand users switch the active portfolio with `portfolio({action:"switch", portfolioId})`.

Flow after `get_result` returns `complete`:

```
publish_post({ apiKey, postId })                                  // no channelIds → returns availableChannels + recommendedSlots
publish_post({ apiKey, postId, channelIds })                      // post NOW (queued; worker publishes within minutes)
publish_post({ apiKey, postId, channelIds, scheduledAt })         // schedule (ISO, future, before the ~7-day video expiry)
```

`postId` comes from the `confirm` response (preferred — survives session expiry); `sessionId` also works while the session is alive. **Always show `availableChannels` and let the USER pick** — never auto-select where their content goes. Re-calling reschedules: already-published channels are preserved, deselected unpublished ones are dropped. The user can move or cancel scheduled posts in the Poppify app calendar.

**Scheduling for engagement/reach:** the no-channelIds response also returns `recommendedSlots` — the account's best posting times from its real engagement history (platform norms for new accounts; hours are UTC). Pass a slot's `nextOccurrence` as `scheduledAt`, or `scheduledAt:"best"` to auto-pick the top slot; scheduling responses include a `slotAssessment` ranking the chosen time. For the full playbook (basis handling, sample-size honesty, multi-post spacing), invoke the **`poppify-schedule-optimizer`** skill.

## Gotchas worth knowing

- **Don't burn seeds on text-heavy slides.** Gemini Imagen garbles literal text (terminal frames, install commands, stat callouts). For those: HTML/CSS screencap locally → `upload_asset` → `update_slides({action:"set_image", slideIndex, imageUrl:accessUrl})`. ZERO seeds.
- **Single-image reel → ONE continuous camera move.** When one image carries all slides, keep a SINGLE `videoEffect` and leave `continuousEffect` on (default `true`). The renderer then makes one continuous move across the whole reel via a global frame offset. **Assigning a DIFFERENT effect per slide on a same-image run disables continuous smoothing** — each slide gets its own independent move and the motion visibly resets at every cut. Only use per-slide `slideEffects` when the slides have *different* images.
- **Composer draws caption text BY DEFAULT** from `slides[i].voiceoverShort`. Empty string = skipped.
- **Slide duration: two DRIVERS + two MEDIA FLOORS, no global control.** Media floors always play in full: (1) attached **voiceover** audio, (2) a rendered **live-motion (Veo) clip** — so an 8s morph slide holds all 8s even under a short caption (you do NOT pad the caption). Drivers: (3) **EXPLICIT** `update_slides({action:"set_duration", slideIndex, duration:N})` on ANY slide — overrides text-length, capped 2–15s, never shortens the media floor; (4) **TEXT-LENGTH** `(words/2.6)*1.2` otherwise; 4s default for a blank slide. Lengthen a slide: write MORE text OR use set_duration. `apply_session_patch({duration})` is rejected — no session-level control. **Use text** for normal captioned slides; **use set_duration** for text-baked cards or any precise hold.
- **Voiceover auto-detaches when text changes.** The text you write IS the voiceover script. `update_slides({action:"set_text"})` on a slide with attached voiceover detaches it (old audio of wrong words). Call `add_narration` ONLY after text is finalized — otherwise you waste 5 seeds per text edit.
- **Text cards (terminal screencaps, end cards)**: `set_image` an image with text already baked in, then `update_slides({action:"set_text", slideIndex, newText:""})`. Empty string suppresses composer text overlay. Hold it as long as you want with `update_slides({action:"set_duration", slideIndex, duration:N})` (2–15s) — set_duration works on any slide, so you no longer have to blank the caption first just to control the hold. Blank slide with no explicit duration falls back to 4s.
- **Sceneboard recipes auto-lock motion across all panels** — per-slide variation is ignored when visualType is sceneboard.
- **Library audio resolves URLs at attach time** — if `apply_session_patch({audio:{source:"library", assetId}})` errors with "asset not found", pick a different `assetId` from `get_music_library`.

## When something looks wrong with the finished video

Invoke the `poppify-troubleshoot` skill. It has the decision tree for common symptoms (wrong colors, missing audio, caption drops, duration mismatch).
