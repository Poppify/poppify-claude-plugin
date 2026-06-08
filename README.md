# Poppify — Claude Code Plugin

> **Photo-led short-form vertical reels for Instagram / TikTok / YouTube Shorts / Facebook Reels.** Upload 1–10 photos, get a captioned 15/30/60s reel with motion, music, and optional voiceover. **$0.06 base render. 50 free seeds on signup. No subscription.**

[Poppify](https://poppify.ai) is a photo-led creative pipeline that *composes* reels (FFmpeg motion + library-matched assets + recipe-driven narrative + on-screen text), not a text-to-video generator. That's why the base render is 1 seed (~$0.06) instead of dollars-per-second like generative video services. When you need standalone AI assets, `generate_image` / `generate_music` / `generate_voiceover` are callable on their own (10 seeds each).

**Not for:** text→video generation (Sora, Veo, Kling — use a generative video MCP), 4K horizontal cinema, or sub-4-second clips.

## What's inside

| Component | Purpose |
|---|---|
| **MCP server** (`https://poppify.ai/mcp`, HTTP) | All Poppify tools: `register`, `start_session_from_photos`, `apply_session_patch`, `customize`, `set_audio`, `generate_image`, `confirm`, `get_result`, etc. |
| **`/poppify:make-reel`** slash command | Guided session flow — photo-led or topic-led entry, free customize loop, paid confirm. |
| **`/poppify:troubleshoot`** slash command | Symptom triage when a render came out wrong. |
| **`/poppify:verify-render`** slash command | Download + ffprobe + frame-extract verdict on a finished MP4. |
| **`poppify-build-reel`** skill | The canonical photo-led / topic-led flow — when to use which tool, where the free vs paid boundaries are. |
| **`poppify-render-debug`** skill | Download the finished MP4, run ffprobe, extract frames, surface a verdict. |
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
- **`generate_image`** (Gemini): 10 seeds per image
- **`generate_music`** (ElevenLabs Music): 10 seeds per track
- **`generate_voiceover`** (ElevenLabs Voice): 10 seeds per batch

Seeds are sold at $5.99 for 100 seeds (standard pack) or $0.50 for 5 seeds (mini trial pack). 50 seeds are granted free on signup.

## Also useful standalone

The reel pipeline is the headline, but every generation primitive is callable on its own — handy for ad-hoc creative tasks, not just full reels:

- **`generate_image`** — Gemini Imagen, 10 seeds. Workshop the prompt for free first via `suggest_image_prompt`, then generate. Returns a URL.
- **`generate_music`** — ElevenLabs Music, 10 seeds. Workshop via `suggest_music_prompt` (free), then generate. Returns a URL.
- **`generate_voiceover`** — ElevenLabs Voice, 10 seeds per batch. Browse the curated voice catalog (free) via `list_voices`, then generate from a script.
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
