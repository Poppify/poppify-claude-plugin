# Poppify — Claude Code Plugin

> Generate cinematic vertical reels directly from Claude Code. One install adds the Poppify MCP server **and** four skills that teach Claude how to use it well.

[Poppify](https://poppify.ai) is an AI creative-video pipeline: photos or topics in, finished 720x1280 reels (with motion, music, captions, voiceover) out. This plugin packages the public Poppify MCP (HTTP) together with the operational knowledge Claude needs to drive it reliably.

## What's inside

| Component | Purpose |
|---|---|
| **MCP server** (`https://poppify.ai/mcp`, HTTP) | All Poppify tools: `register`, `start_session_from_photos`, `apply_session_patch`, `customize`, `set_audio`, `generate_image`, `confirm`, `get_result`, etc. |
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
