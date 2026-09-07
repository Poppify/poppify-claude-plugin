# Poppify — AI Reels Maker & Video Generator (MCP Server + Claude Code Plugin)

> **Make reels worth sharing, from inside Claude.** The AI video generator and reels maker for Instagram, TikTok, YouTube Shorts and Facebook. Give it 1–10 photos or just a topic and get back a captioned 9:16 reel — **Live Motion** that animates the subject inside a still, AI-generated scenes you don't already have, cinematic camera moves, AI voiceover and music, assembled into one edit and published to your channels. **$0.06 base render. 50 free seeds on signup, up to 150 once you connect a social account. No subscription.**

### What you can make

- **Live Motion** — image-to-video animates the subject inside a still (breath, blink, a turn of the head) while the camera move layers on top. ~10 seeds; cache hits are free.
- **Images & shots** — generate the scenes you don't already have, or build the reel around your own photos.
- **Music & voiceover** — a soundtrack and AI narration, mixed with ducking and start-cue alignment.
- **The edit** — frames, captions, music and voice assembled into one reel. This is the step that makes a reel rather than a pile of assets.
- **Share on social** — publish straight to Instagram, TikTok, YouTube Shorts or Facebook. Publishing is free.

[Poppify](https://poppify.ai) is an AI video generator built for reels specifically. It **composes** — FFmpeg motion, library-first asset matching, recipe-driven narrative and on-screen text — rather than hallucinating every frame, which is why a finished reel is ~$0.06 instead of dollars-per-second. Text-in works too: start from a topic with no photos at all and it generates the scenes.

**Where it fits in a Claude Code stack:** pairs with [Postiz](https://github.com/gitroomhq/postiz-agent) for cross-platform scheduling and [Windsor.ai](https://github.com/windsor-ai/claude-windsor-ai-plugin) for attribution. Use [HyperFrames](https://github.com/heygen-com/hyperframes) when you want to code video in HTML.

**Not for:** avatar-based presenter video (use HeyGen / Synthesia), 4K horizontal cinema, or sub-4-second clips.

**Built for:** anyone who posts every day and wants the reel finished rather than started.

## What's inside

| Component | Purpose |
|---|---|
| **MCP server** (`https://poppify.ai/mcp`, HTTP) | All Poppify tools: `register`, `start_session_from_photos`, `apply_session_patch`, `update_slides`, `add_slide_image`, `confirm`, `get_result`, `publish_post` (post now / schedule to connected channels, linked accounts), `portfolio` (list / switch), etc. |
| **`/poppify:make-reel`** slash command | Guided session flow — photo-led or topic-led entry, free customization loop, paid confirm. |
| **`/poppify:troubleshoot`** slash command | Symptom triage when a render came out wrong. |
| **`/poppify:verify-render`** slash command | Optional (shell + ffmpeg): download + ffprobe + frame-extract verdict on a finished MP4. |
| **`poppify-build-reel`** skill | The canonical photo-led / topic-led flow — when to use which tool, where the free vs paid boundaries are. |
| **`poppify-text-card`** skill | Render a pixel-perfect text card (terminal frame, code, stat callout, headline) locally via HTML/CSS + a headless browser, cross-platform. Shell-capable clients only; shell-less clients let the composer draw the caption. |
| **`poppify-live-motion`** skill | Veo live-motion prompting playbook — plain subject animation AND first/last-frame transitions (composition-locked end frames, journey overridePrompt, failure modes). |
| **`poppify-render-debug`** skill | Optional (shell + ffmpeg): download the finished MP4, run ffprobe, extract frames, surface a verdict. |
| **`poppify-troubleshoot`** skill | Decision tree: symptom → root cause → action for missing audio, wrong colors, dropped captions, stuck renders. |
| **`poppify-schema-introspect`** skill | How to verify the deployed MCP schema when a parameter looks silently dropped. |
| **`poppify-schedule-optimizer`** skill | Engagement-optimized publishing — `recommendedSlots` (real engagement history vs platform norms), `scheduledAt:"best"`, slot assessment, multi-post spacing. |

## Install

In Claude Code:

```
/plugin marketplace add Poppify/poppify-claude-plugin
/plugin install poppify@poppify
```

That's it. After install:

1. `register()` (optionally `{ label: "claude" }`) mints a wallet and returns an `apiKey` + a `signupBonusUrl` (50 free seeds, up to 150 after connecting a social account — claim before paying).
2. Use the `poppify-build-reel` skill (Claude will auto-invoke when you ask it to "make a reel") to drive the rest.

## What does it cost?

- **MCP install**: free
- **All non-generation tools** (search library, apply session patches, attach audio, update slides, register, wallet balance / top-up / link, portfolio list/switch, publish + schedule): **free**
- **`confirm` render**: 1 seed base
- **`add_slide_image`** (Gemini): 5 seeds per image
- **`add_soundtrack`** (ElevenLabs Music): 5 seeds per track
- **`add_narration`** (ElevenLabs Voice): 5 seeds per batch
- **`animate_slide`** (image-to-video, OPTIONAL): 10 seeds per live clip. Animates the subject *inside* a still (blink, breath, micro-gesture) while the FFmpeg camera motion layers on top. A per-slide upgrade applied **after** you review the cinematic baseline — never by default. Cache hits via `search_live_library` are free.

Seeds are sold at $5.99 for 100 seeds (standard pack) or $0.50 for 5 seeds (mini trial pack). 50 seeds are granted free on signup, and up to 150 once you connect a social account.

## Generation as a workflow step

Generation is never the destination — it fills in what the library can't, *inside* a reel session. Always search the library first (free); generate only when there's no good match:

- **`search_visual_library`** / **`get_music_library`** — search the existing royalty-free library first. FREE. Matches at score ≥ 40 generally beat fresh AI generation.
- **`add_slide_image`** — fill a slide with a Gemini Imagen still, 5 seeds. Workshop the prompt free via `suggest_prompt({kind:"image"})` first.
- **`add_soundtrack`** — add an ElevenLabs Music track to the reel, 5 seeds. Workshop via `suggest_prompt({kind:"music"})` (free) first.
- **`add_narration`** — narrate slides via ElevenLabs Voice, 5 seeds per batch. Pick a voice free via `list_voices` first.

Generated assets persist to your asset library, so they're reusable across future reels and discoverable on later searches.

## Manual install (without the plugin)

If you only want the MCP and don't want the skills:

```
claude mcp add --transport http poppify https://poppify.ai/mcp
```

You'll get the same tool surface, but Claude won't have the pre-loaded knowledge about when to call which tool or how to verify the finished video.

## Updates

The plugin uses semantic versioning. To get updates:

```
/plugin marketplace update poppify
```

Skill content evolves as the MCP surface evolves — we publish new versions when a tool surface changes meaningfully.

## Support

- **Bug reports / feature requests**: file via `submit_feedback({ apiKey, ... })` from inside any Claude session — the feedback flows directly to the Poppify team's triage queue.
- **GitHub Issues**: https://github.com/Poppify/poppify-claude-plugin/issues
- **Web**: https://poppify.ai

## License

MIT — see [LICENSE](./LICENSE).
