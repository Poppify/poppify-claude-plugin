# Poppify — Photo to TikTok & Instagram Reels (MCP Server + Claude Code Plugin)

> **MCP server and Claude Code plugin for photo to TikTok, Instagram Reels, YouTube Shorts, and Facebook video.** Upload 1–10 photos, get a captioned 15/30/60s reel with motion, library-matched music, and optional voiceover. **$0.06 base render. 50 free seeds on signup. No subscription.**

[Poppify](https://poppify.ai) is a Claude Code plugin (and standalone MCP server) that *composes* reels via a photo-led creative pipeline (FFmpeg motion + library-first asset matching + recipe-driven narrative + on-screen text), not a text-to-video generator. That's why the base render is 1 seed (~$0.06) instead of dollars-per-second like generative video services (Runway, Sora, Kling, Vidu, Veo). When you need standalone AI assets, `generate_image` / `generate_music` / `generate_voiceover` are callable on their own (5 seeds each).

**Where it fits in a Claude Code marketing stack:** Poppify is the creative slot. Pairs with [Postiz](https://github.com/gitroomhq/postiz-agent) for cross-platform scheduling and [Windsor.ai](https://github.com/windsor-ai/claude-windsor-ai-plugin) for attribution. Drop-in replacement for [Runway](https://github.com/runwayml/skills) when you have photos and want library-matched audio; use [HyperFrames](https://github.com/heygen-com/hyperframes) when you want to code video in HTML.

**Not for:** text→video generation (Sora, Veo, Kling — use Runway or a generative video MCP), avatar-based video (use HeyGen / Synthesia), 4K horizontal cinema, or sub-4-second clips.

**Built for:** SMBs (5–19 employees) and solo service providers who want consistent vertical reels without hiring a content creator. The agency replacement at $30–60/mo instead of $3K+.

## What's inside

| Component | Purpose |
|---|---|
| **MCP server** (`https://poppify.ai/mcp`, HTTP) | All Poppify tools: `register`, `start_session_from_photos`, `apply_session_patch`, `customize`, `set_audio`, `generate_image`, `generate_frames`, `confirm`, `get_result`, etc. |
| **`/poppify:make-reel`** slash command | Guided session flow — photo-led or topic-led entry, free customize loop, paid confirm. |
| **`/poppify:troubleshoot`** slash command | Symptom triage when a render came out wrong. |
| **`/poppify:verify-render`** slash command | Optional (shell + ffmpeg): download + ffprobe + frame-extract verdict on a finished MP4. |
| **`poppify-build-reel`** skill | The canonical photo-led / topic-led flow — when to use which tool, where the free vs paid boundaries are. |
| **`poppify-text-card`** skill | Render a pixel-perfect text card (terminal frame, code, stat callout, headline) locally via HTML/CSS + a headless browser, cross-platform. Shell-capable clients only; shell-less clients let the composer draw the caption. |
| **`poppify-render-debug`** skill | Optional (shell + ffmpeg): download the finished MP4, run ffprobe, extract frames, surface a verdict. |
| **`poppify-troubleshoot`** skill | Decision tree: symptom → root cause → action for missing audio, wrong colors, dropped captions, stuck renders. |
| **`poppify-schema-introspect`** skill | How to verify the deployed MCP schema when a parameter looks silently dropped. |

## Install

In Claude Code:

```
/plugin marketplace add Poppify/poppify-claude-plugin
/plugin install poppify@poppify
```

That's it. After install:

1. `register({ source: "claude" })` mints a wallet and returns an `apiKey` + a `signupBonusUrl` (50 free seeds — claim it before paying).
2. Use the `poppify-build-reel` skill (Claude will auto-invoke when you ask it to "make a reel") to drive the rest.

## What does it cost?

- **MCP install**: free
- **All non-generation tools** (search library, customize, attach audio, update slides, register, get balance, get topup URL): **free**
- **`confirm` render**: 1 seed base
- **`generate_image`** (Gemini): 5 seeds per image
- **`generate_music`** (ElevenLabs Music): 5 seeds per track
- **`generate_voiceover`** (ElevenLabs Voice): 5 seeds per batch
- **`generate_live_motion`** (Veo 3.1 Lite image-to-video, OPTIONAL): 10 seeds per live clip. Animates the subject *inside* a still (blink, breath, micro-gesture) while the FFmpeg camera motion layers on top. A per-slide upgrade applied **after** you review the cinematic baseline — never by default. Cache hits via `search_live_library` are free.

Seeds are sold at $5.99 for 100 seeds (standard pack) or $0.50 for 5 seeds (mini trial pack). 50 seeds are granted free on signup.

## Also useful standalone

The reel pipeline is the headline, but every generation primitive is callable on its own — handy for ad-hoc creative tasks, not just full reels:

- **`generate_image`** — Gemini Imagen, 5 seeds. Workshop the prompt for free first via `suggest_image_prompt`, then generate. Returns a URL.
- **`generate_music`** — ElevenLabs Music, 5 seeds. Workshop via `suggest_music_prompt` (free), then generate. Returns a URL.
- **`generate_voiceover`** — ElevenLabs Voice, 5 seeds per batch. Browse the curated voice catalog (free) via `list_voices`, then generate from a script.
- **`search_visual_library`** / **`get_music_library`** — search the existing royalty-free library before generating. FREE. Matches at score ≥ 40 generally beat fresh AI generation.

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
