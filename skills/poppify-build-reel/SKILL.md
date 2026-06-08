---
name: poppify-build-reel
description: Step-by-step guide for generating a cinematic vertical reel via the Poppify MCP — covers the photo-led and topic-led entries, the free customize loop, and the single paid confirm step. Use when the user asks "make me a reel", "turn these photos into a video", or anything similar.
---

# Building a Poppify reel — the canonical flow

Poppify is FREE for everything except `confirm` (1 seed = $0.006 base render) and `generate_*` tools (10 seeds each for AI image / music / voiceover). You can iterate the configuration **infinitely** before render and not spend anything.

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

Returns 5 concept options. Pick one with `refine_concept`.

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
  duration: "30s",                         // 15s | 30s | 60s
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
- `customize(...)` — production knobs
- `set_audio(...)` — attach music
- `update_visual(...)` — per-slide image swap or insert

**Search the library FIRST** before generating:
- `search_visual_library({ apiKey, query, limit })` for images — score ≥ 40 should beat AI gen
- `get_music_library({ apiKey, mood, genre })` for music

## Step 4 — Generate AI assets only when needed

These cost 10 seeds each. Always workshop the prompt for free first:

```
suggest_image_prompt({ apiKey, slideContext })   // FREE prompt refinement
generate_image({ apiKey, prompt, style })        // 10 seeds, returns image URL
// Then attach via apply_session_patch.visualEdits or update_visual

suggest_music_prompt({ apiKey, mood, genre })    // FREE prompt refinement
generate_music({ apiKey, prompt, durationSeconds }) // 10 seeds, returns URL
// Then attach via apply_session_patch.audio or set_audio

list_voices({ apiKey })                          // FREE voice catalog
generate_voiceover({ apiKey, scripts, voiceId }) // 10 seeds per batch
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

## Gotchas worth knowing

- **Don't burn seeds on text-heavy slides.** Gemini Imagen garbles literal text (terminal frames, install commands, stat callouts). For those: HTML/CSS screencap locally → `upload_asset` → `update_visual({source:"user_url",url:accessUrl})`. ZERO seeds.
- **Composer draws caption text BY DEFAULT** from `slides[i].voiceoverShort`. Empty string = skipped.
- **Text-length drives slide duration.** To make a slide last longer, write MORE text on it via `update_slides set_text`. There's no `duration` knob per slide.
- **Sceneboard recipes auto-lock motion across all panels** — per-slide variation is ignored when visualType is sceneboard.
- **Library audio resolves URLs at attach time** — if `set_audio({source:"library"})` errors with "asset not found", pick a different `assetId` from `get_music_library`.

## When something looks wrong with the finished video

Invoke the `poppify-troubleshoot` skill. It has the decision tree for common symptoms (wrong colors, missing audio, caption drops, duration mismatch).
