---
name: poppify-build-reel
description: Canonical photo-led / topic-led flow for the Poppify MCP. Use whenever the user wants: a reel, short video, vertical video, Instagram reel, TikTok video, TikTok, YouTube Short, YouTube Shorts, Facebook reel, FB reel, 15-second video, 30-second video, 60-second video, photo slideshow, photo to video, animate photos, photo animation, slideshow video, social media video, content for social, brand video, product reel, ad creative for social, before/after reel, transformation video, story reel, hook video, or any vertical short-form video for IG/TikTok/YT/FB. Covers the free customization loop and the single paid confirm step. Base render = 1 seed (~$0.06). 50 free seeds on signup, up to 150 after connecting a social account. Use poppify-troubleshoot if the render comes out wrong; poppify-render-debug to verify the finished MP4.
---

# Building a Poppify reel — the canonical flow

Poppify is FREE for everything except `confirm` (1 seed base render) and the `generate_*` tools (5 seeds each for AI image / music / voiceover; `animate_slide` is 10 seeds per clip). You can iterate the configuration **infinitely** before render and not spend anything.

> **Tier awareness — check what's doable first.** Not every Poppify connector exposes AI generation. Before using `add_slide_image`, `animate_slide`, `add_soundtrack`, `add_narration`, or `set_narration_voice`, confirm they exist — call `get_capabilities` (it reports only what *this* tier can do) or check the tool list. If they're absent (the **composer tier**), build entirely from the user's own photos + `search_visual_library` / `search_live_library` / `get_music_library`; never assume a generator exists.

> **Work from the DECK VIEW, by asset id — never by URL.** Every session response ends with a deck view: the shot list in order, each shot's asset id and filename, its flags, the music, what is *not* in the reel, and a version. Every piece of media the session has seen has a short, stable id — `P` photo, `V` uploaded video, `L` live clip, `G` generated image, `T` text card, `M` music, `N` narration. A deck view looks like this (abridged):
>
> ```
> DECK v7 — 4 shots, ~39.3s. #n = slideIndex. Edit by asset id; pass expectedVersion:7.
> #0 P4 reels1.jpg [cover · pin:first · no-caption · whole] 3.0s
> #1 P1 IMG_1001.jpg "Autumn in the garden" 3.5s
> #2 P3 IMG_1003.jpg large:L2/2 "Book a consultation" 4.5s
> music: M2 «Quiet Luxury» rendered v5
> not in reel: P5 example.png reference_only; M1 auto-picked
> ```
>
> - **Address media and shots by id** (`asset:"P4"`, `audio:{assetId:"M2"}`), not by signed URL — URLs rotate on every read and re-pasting them is how shots get swapped by accident.
> - **Pass `expectedVersion`** (the N in "DECK vN") on every edit. If the deck changed since you read it, the edit is refused with `deck_stale` and nothing is applied — you get the current deck back to re-decide against. Your own earlier edits in the same turn never make the next one stale. Omitting it still works, with a warning.
> - **Send only what changes.** A music-only turn sends `audio`, never `slides[]`; moving one shot is `update_slides({action:"move", asset:"P4", to:"first"})`, not a re-sent deck. Several edits → `update_slides({ops:[...]})` in one call.

## Step 1 — Mint a wallet (one-time, free)

If the user hasn't registered yet:

```
register()                       // optional: { label: "claude" } for provenance
```

Returns `apiKey` and `signupBonusUrl`. **Surface the signupBonusUrl** — opening it and signing in with Google grants 50 free seeds — and connecting a social account earns up to 150 total (≈ 50 base renders, or ~3 fully-loaded reels WITH AI image + AI music + AI voiceover at ~16 seeds each). Don't ask the user to pay before they've claimed this.

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
  aspectRatio: "9:16" | "16:9"   // OPTIONAL — otherwise the photos decide. See below.
})
```

Returns: slides[] (each with `voiceoverShort` text), caption, hashtags, callToAction, picked recipe.

**Topic-led** (no photos, brand or topic input):
```
start_session_from_topic({ apiKey, topic, audience, goal, aspectRatio })
// NOTE: no `platform` here — the topic schema does not declare it, so it is
// silently dropped. Only start_session_from_photos accepts it.
```

**Aspect ratio is a SESSION property.** Photo-led sessions take it from the photos
(landscape photos → 16:9) unless you pass `aspectRatio`; topic-led sessions default to
`9:16`. Everything downstream inherits it: `add_slide_image` stills, live-motion clips,
library search/match, and the render canvas. If the user later asks to change it ("make it
vertical for mobile"), **change it** with `apply_session_patch({sessionId, aspectRatio:"9:16"})`
— the response echoes the stored value and the crop it costs (`orientation.note`); relay that
before they pay. Never tell the user it changed without that call. Don't mix orientations: a
16:9 still in a 9:16 session gets cropped/letterboxed, and `add_slide_image` will warn.

Returns 5 concept options. Pick one with `refine_concept`. **A topic-led session starts with NO images** — every slide's `imageUrl` is empty. Every shot needs its own visual before `confirm` — otherwise confirm refuses BEFORE charging (`slides_missing_images`, naming the empty shots). The free fixes come first: remove the extra beats, or place one of their photos / a library image. The `recipeDistribution` field shows which recipes the 5 concepts span; if they collapsed off-brief (e.g. you wanted a tribute/hype angle but got all confessional), call `start_session_from_topic` again with an explicit `recipe` parameter (`recipes()` for IDs).

### Decide FIRST: one image, or one per beat?

Before generating anything, ask whether **one image can carry the whole reel**. For a single-subject / hero / cinematic reel (recipe `visualType: "hero_image"`), it usually can — the camera motion and the changing captions ARE the variety; the image stays the star. Generating four images for four beats is the wrong default — it costs 4× the seeds AND drifts the subject's identity across slides.

- **One image across all beats** (recommended for single-subject reels): `add_slide_image` once, then `set_image` the SAME asset id on every slide. Keep ONE `videoEffect` + `continuousEffect` on so the camera makes one continuous move (see Gotchas). 5 seeds total for imagery.
- **One image per beat** (only when beats are genuinely distinct scenes — before/after, multi-step, comparison): generate/search per slide. N × 5 seeds.

## Step 3 — Refine (FREE, unlimited iteration)

The single most efficient call here is **`apply_session_patch`** — it batches per-slide text + music + visual edits + production knobs + textColor + per-slide motion into ONE round-trip:

```
apply_session_patch({
  sessionId,
  expectedVersion: 7,                      // the N in "DECK vN" — see the deck-view note above
  // Per-slide caption text. Setting text is what turns a slide's caption ON
  // (captions are opt-in); empty string = a shot left blank ON PURPOSE.
  slides: [
    { index: 0, text: "Your literal slide 1 caption" },
    { index: 1, text: "" },               // deliberately no caption (e.g. text baked into the image)
    { index: 2, text: "Book a consultation\nexample.com", heroLine: 1 }  // which line renders LARGE (1-based)
  ],
  // Production knobs
  aspectRatio: "9:16",                     // change orientation mid-session (response states the crop)
  textColor: "#FFFFFF",                    // hex; overrides recipe palette
  accentColor: "#88beb4",                  // second brand colour: the rule / highlighted word
  textAnimation: "editorial",              // editorial | lower_third | karaoke (position is baked in)
  font: "Georgia",                         // a font the USER named → nearest caption style; relay font.note
  videoEffect: "push_in",                  // session-wide motion (canonical: push_in, pull_out, lateral_pan, vertical_pan, focus_pull, epic_parallax, static)
  audioMood: "uplifting",
  // NOTE: no SESSION-level `duration` knob. Per-slide length is resolved as:
  // media floors (voiceover audio + a rendered live-motion clip) ALWAYS play in
  // full; then explicit set_duration (any slide, capped 2–15s) or text-length
  // drive the rest; 4s default. To change runtime: write more/fewer words, use
  // set_duration on any slide, or add/remove slides.
  // Per-slide motion overrides (takes precedence over session-wide — but is IGNORED entirely while `continuousEffect:true`, which drives one curve across the whole run)
  slideEffects: [
    { slideIndex: 3, videoEffect: "focus_pull" }  // end card → minimal motion
  ],
  // Music attach (free — library audio is resolved to a playable URL here)
  audio: { source: "library", assetId: "..." }
})
```

If you'd rather iterate one knob at a time, the per-action tools all work (pass `expectedVersion` on each):
- `update_slides({ action: "set_text", asset: "P1", newText })` — literal caption on the shot showing P1 (or `slideIndex`)
- `update_slides({ action: "set_image", slideIndex, asset: "P4" })` — put a registered asset on a shot
- `update_slides({ action: "move", asset: "P4", to: "first" })` — move ONE shot; everything else keeps its order
- `update_slides({ ops: [...] })` — several edits in one all-or-nothing call (indexes = the deck as it is now)
- `apply_session_patch({ audio: { source: "library", assetId: "M2" } })` — music by id (library ids work too)
- `apply_session_patch({...})` — production knobs (one knob per call works fine)
- `apply_session_patch({visualEdits:[...]})` — insert/splice slides

**Search the library FIRST** before generating:
- `list_assets({ apiKey, limit })` FIRST — the user's own library: every image, text card and Live Motion clip they already paid for. Reusing one is free. `visualType:"live_motion"` narrows to clips; `cursor`/`nextCursor` page through a big library. Spending 5 seeds to regenerate something they already own is the most common avoidable cost in the product.
- `search_visual_library({ apiKey, keywords, limit })` for images (keywords = string or array: subject + mood + scene) — score ≥ 40 should beat AI gen
- `get_music_library({ apiKey, mood, genre })` for music

### Record what the user says about their assets (FREE) — the server holds you to it

When the user says what something IS or how it must be treated, write it down with
`assets({action:"set"})` the moment they say it. The server then enforces it on every later
edit and before `confirm` — so a later turn cannot quietly undo it:

```
assets({ sessionId, expectedVersion, action: "set", asset: "P4", attrs: { role: "cover" } })
//   "reels1.jpg is the cover" → pinned first, no caption, and no edit may move it or add text
assets({ sessionId, action: "set", asset: "P5", attrs: { role: "reference_only" } })
//   "that screenshot is only an example" → it can never be placed as a shot
assets({ sessionId, action: "set", asset: "P4", attrs: { fit: "whole" } })
//   "don't crop it / show the whole image" → rendered uncropped (banded), not cover-cropped
assets({ sessionId, action: "reject", asset: "M3" })
//   "not that track" → it cannot come back until she asks for it
assets({ sessionId, action: "set", asset: "M2", attrs: { label: "first track" } })
//   your note in her words, so "that one" can be found again
```

- **Tightening is yours to record** (pin, reject, cover, reference_only, `caption:none`, `fit:whole`). **Loosening is hers** (unpin, restore, text back on a no-caption shot, `fit` away from whole): the refusal carries a `question` to put to the user; third-party clients pass `userAsked:true` only when the user asked for it in their own words.
- **"The first music / the original photo"** is the one with the EARLIEST render (`rendered vN` in the deck view), not the lowest id — an `auto-picked` asset that was replaced before anything rendered was never heard by the user.
- **Her logo** goes to her brand profile with `portfolio({action:"set_logo", sessionId, asset:"P9"})`. It changes her account, so it asks her first. The renderer stamps it on every reel as a **small round badge in the bottom-right corner** (a wide logo is cropped to its centre) — tell her exactly that; it is not a shot and cannot be moved or resized. Don't rebuild it as a text card.
- **A font she names** → `apply_session_patch({font:"Georgia"})`. Custom fonts aren't supported: it maps to the nearest caption style and `font.note` says which — relay it plainly, never as "Georgia (or close to it)".
- **A copy question** (`copy_replacement_needs_confirmation`): the rest of the patch applies and only the shot texts wait (`partiallyApplied`, `pending.lines`). Tell her both halves — never describe the pending lines as already on the reel.

## Step 4 — Generate AI assets only when needed

These cost 5 seeds each. Always workshop the prompt for free first:

```
suggest_prompt({ apiKey, kind: "image", sessionId, slideIndex })  // FREE — pass sessionId+slideIndex
                                                 //   for the slide's composer plan (or subjectDescription when no session)
add_slide_image({ apiKey, prompt, visualStyle, sessionId })  // 5 seeds, returns image URL

// PHOTO-LED SESSIONS GATE THIS. If the session has uploaded photos, a
// reference-free call is refused BEFORE charging, with `reference_required`.
// Pass one of:
//   referenceAssetIds: [...]     // up to 8 (compose: 2-3; collage: all of them), from list_assets / the session
//   referenceImageUrls: [...]    // up to 8
//   referenceSlide: <index>      // reuse a slide's existing image as reference
//   ignoreSessionPhotos: true    // deliberately generate unrelated to their photos
// so the generated scene matches the user's own shots instead of drifting.
// allowTextInImage: true only when you genuinely want baked-in typography —
// otherwise use add_text_card, which is exact and free.
//   Pass sessionId so the still inherits the session's aspect ratio (vertical by
//   default). The render canvas is fixed to the session aspect, so a still that
//   doesn't match it is cropped — passing sessionId keeps them aligned.
// The new image is registered in the session with an id (G1, G2 … — see the deck view).
// Attach via update_slides({action:"set_image", slideIndex, asset:"G1"})
//   — or update_slides({ops:[...]}) / apply_session_patch({slides:[{index, imageUrl:"G1"}]}) for several beats.
//   For a single-image reel, set_image the SAME id on every slide.

suggest_prompt({ apiKey, kind: "music", userInput })     // FREE — userInput = natural-language music description
add_soundtrack({ apiKey, prompt, durationSeconds }) // 5 seeds, returns URL (default 30s, max 300s)
// Then attach via apply_session_patch({audio:{source:"user_url", url}})

list_voices({ apiKey })                          // FREE voice catalog
add_narration({ apiKey, scripts, voiceId }) // 5 seeds per batch
// Then attach via apply_session_patch({voiceoverSlides})
```

## Step 5 — Verify, then confirm

Use `get_result({ sessionId })` to see the seed total — pre-confirm it returns the exact price breakdown. **Render only when the user asked for it in this message, or said yes to the price you just quoted.** A correction, a question or a complaint is never a go-ahead: fix or answer it, say in one line what changed, and ask "Render now? (1 seed)". If they said "don't render", don't end your reply by inviting one either. Then:

```
confirm({ sessionId, apiKey, buyerEmail })
```

Charges seeds. Render typically completes in 30–120s. Poll:

```
get_result({ sessionId, apiKey })
```

When `status === "complete"`, you get a `videoUrl` (signed GCS URL, valid ~7 days — `videoUrlExpiresAtIso` in the response has the exact expiry). Hand it to the user right away and tell them to save it before it expires; on shell clients also `curl` a local copy as the durable backup. Don't tell the user "we emailed it" — Poppify does not email rendered reels; the URL is the only delivery surface.

## Step 6 — OPTIONAL: Live Motion (a per-slide upgrade, only AFTER baseline review)

Live Motion animates the **subject inside a still** (blink, breath, micro-gesture, a small action) using image-to-video, while the FFmpeg camera motion layers on top. It is an OPTIONAL upgrade, NOT a default — the flow is **render the cinematic baseline → user reviews → optionally upgrade selected slides to live**. Never apply it before the user has seen the cinematic render; you'd risk burning seeds on a reel they might already love.

> **For before/after transformations (`endFrameUri` first/last-frame interpolation), invoke the `poppify-live-motion` skill first** — bridged renders need a composition-locked end frame and an authored journey `overridePrompt`; the auto prompt is wrong for that mode.

When the user wants it (same order as the server instructions' lifecycle):
1. `suggest_live_action({ sessionId, slideIndex })` (FREE) → pick a motion verb with the user.
2. `animate_slide({ sessionId, slideIndex, dryRun:true, liveAction })` (FREE) — preview the exact Veo prompt; iterate until it reads right.
3. `search_live_library({ apiKey, imageHash?, actionKeywords:<finalized action>, durationSeconds })` — cache hits (score ≥ 60) attach for **zero seeds**.
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
publish_post({ apiKey, postId, channelIds })                      // PREVIEW — returns status:"confirm_required", publishes nothing
publish_post({ apiKey, postId, channelIds, confirmed:true })      // post NOW (queued; worker publishes within minutes)
publish_post({ apiKey, postId, channelIds, scheduledAt, confirmed:true })  // schedule (ISO, future, before the ~7-day video expiry)
```

**`confirmed:true` is required or nothing is published.** A call without it
returns `status:"confirm_required"` and a preview of the caption, channels and
time — that is the design, not an error. Show the user that preview, get a
real yes, and only then repeat the SAME call with `confirmed:true`. Never send
both in one breath: a wasted seed can be refunded, a wrong caption in front of
an audience cannot. If you find yourself calling this repeatedly and getting
`confirm_required` each time, you are missing the flag — do not report success.

`postId` comes from the `confirm` response (preferred — survives session expiry); `sessionId` also works while the session is alive. **Always show `availableChannels` and let the USER pick** — never auto-select where their content goes. Re-calling reschedules: already-published channels are preserved, deselected unpublished ones are dropped. The user can move or cancel scheduled posts in the Poppify app calendar.

**Scheduling for engagement/reach:** the no-channelIds response also returns `recommendedSlots` — the account's best posting times from its real engagement history (platform norms for new accounts; hours are UTC). Pass a slot's `nextOccurrence` as `scheduledAt`, or `scheduledAt:"best"` to auto-pick the top slot; scheduling responses include a `slotAssessment` ranking the chosen time. For the full playbook (basis handling, sample-size honesty, multi-post spacing), invoke the **`poppify-schedule-optimizer`** skill.

## Gotchas worth knowing

- **Don't burn seeds on text-heavy slides.** Gemini Imagen garbles literal text (terminal frames, install commands, stat callouts). For those call `add_text_card({apiKey, sessionId, slideIndex, text})` — server-side, ZERO seeds, works in every client including shell-less ones. Only when the design needs pixel control (terminal frame, multi-line code) fall back to the local render in the **`poppify-text-card`** skill.
- **Single-image reel → ONE continuous camera move.** When one image carries all slides, keep a SINGLE `videoEffect` and set `continuousEffect: true` (default `false` — you must set it). The renderer then makes one continuous move across the whole reel via a global frame offset. **Assigning a DIFFERENT effect per slide on a same-image run disables continuous smoothing** — each slide gets its own independent move and the motion visibly resets at every cut. Only use per-slide `slideEffects` when the slides have *different* images.
- **On-screen captions are OPT-IN.** The slide text a session starts with is the spoken script; nothing is drawn on the picture until `update_slides({action:"set_text"})` / `apply_session_patch({slides:[{index,text}]})` sets it. A deck with a script and no captions renders wordless — set the text you want shown. Empty string = this shot left blank on purpose (allowed even when every other shot is captioned).
- **Slide duration: two DRIVERS + two MEDIA FLOORS, no global control.** Media floors always play in full: (1) attached **voiceover** audio, (2) a rendered **live-motion (Veo) clip** — so an 8s morph slide holds all 8s even under a short caption (you do NOT pad the caption). Drivers: (3) **EXPLICIT** `update_slides({action:"set_duration", slideIndex, duration:N})` on ANY slide — overrides text-length, capped 2–15s, never shortens the media floor; (4) **TEXT-LENGTH** `(words/2.6)*1.2` otherwise; 4s default for a blank slide. Lengthen a slide: write MORE text OR use set_duration. `apply_session_patch({duration})` is silently STRIPPED (not in the schema, so no error comes back) — no session-level control. **Use text** for normal captioned slides; **use set_duration** for text-baked cards or any precise hold.
- **Voiceover auto-detaches when text changes.** The text you write IS the voiceover script. `update_slides({action:"set_text"})` on a slide with attached voiceover detaches it (old audio of wrong words). Call `add_narration` ONLY after text is finalized — otherwise you waste 5 seeds per text edit.
- **Images with text already baked in (her cover, terminal screencaps, end cards)**: place the image, then record `assets({action:"set", asset, attrs:{caption:"none"}})` (or `role:"cover"`, which implies it) — or at least `set_text` with `newText:""`. Never put placeholder text (a space, a dot, a slogan) on a shot the user said to leave blank. Hold it as long as you want with `update_slides({action:"set_duration", slideIndex, duration:N})` (2–15s) — set_duration works on any slide, so you no longer have to blank the caption first just to control the hold. Blank slide with no explicit duration falls back to 4s.
- **Sceneboard recipes auto-lock motion across all panels** — per-slide variation is ignored when visualType is sceneboard.
- **Library audio resolves URLs at attach time** — if `apply_session_patch({audio:{source:"library", assetId}})` errors with "asset not found", pick a different `assetId` from `get_music_library`. Once a track has been in the session it has an `M` id — put it back with `audio:{assetId:"M2"}`.
- **Which caption line renders large** (editorial / lower_third): the renderer picks the hero line itself and never picks a URL, domain or @handle. When the user says which line should be big ("CTA large, site smaller"), set `heroLine` on that slide; the deck view shows `large:Ln/m`. Describe sizes only as the deck view shows them.

## When something looks wrong with the finished video

Invoke the `poppify-troubleshoot` skill. It has the decision tree for common symptoms (wrong colors, missing audio, caption drops, duration mismatch).
