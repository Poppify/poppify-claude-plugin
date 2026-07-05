---
name: poppify-schema-introspect
description: Inspect the deployed Poppify MCP tool schemas to verify what parameters are actually accepted. Use when: a parameter you passed seems to have been silently dropped, an apply_session_patch field didn't take effect, the render came out using defaults instead of your overrides, OR before recommending a parameter you're not 100% sure is in the current deployed build. Diffs schema-vs-docs and helps file precise feedback when drift is found.
---

# Inspecting the deployed Poppify MCP schemas

MCP clients strip undeclared fields before forwarding to the server. If a parameter you pass to a tool seems to have been ignored, the most likely cause is **schema/code drift** — the server's handler accepts the field, but the MCP tool registration's Zod schema doesn't declare it, so the field was dropped at the protocol boundary.

This skill teaches Claude how to verify the deployed schema directly.

> **Jul 2026 consolidation note:** the deployed surface was consolidated from 37 tools to 28 — old names were merged, not dropped, and two NEW tools were added: `publish_post` (post now / schedule to connected channels, linked accounts only) and `portfolio` (list / switch the active portfolio). `customize` / `set_audio` / `set_voiceover` → `apply_session_patch`; `update_visual` → `update_slides({action:"set_image"})` (replace) + `apply_session_patch({visualEdits:[...]})` (insert/splice); `preview_live_prompt` → `generate_live_motion({dryRun:true})`; `generate_frames` → `generate_image` (reference params, consistent-frames mode); `list_recipes` / `get_recipe_options` → `recipes`; `get_balance` / `get_topup_url` / `link` → `wallet`; `suggest_image_prompt` / `suggest_music_prompt` → `suggest_prompt({kind})`; `get_price` → `get_result` (pre-confirm price breakdown). If a schema-vs-docs diff shows one of the old names missing from the live catalog, that's the consolidation — not drift.

## Where to look

The MCP server publishes its tool catalog via the standard MCP `tools/list` request. Claude Code already calls this on connect — the tool definitions are available in the client's MCP state.

To inspect what was returned:

1. List the tools Claude Code currently sees:
   ```
   /mcp poppify
   ```
   That shows the names, but not the param shapes.

2. For the param shapes, ask the deployed server directly via JSON-RPC `tools/list` at the canonical MCP endpoint (`https://poppify.ai/api/mcp`, POST — the Poppify server is stateless, no initialize handshake needed):
   ```bash
   curl -s https://poppify.ai/api/mcp \
     -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
     -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
     | sed -n 's/^data: //p' | jq '.result.tools[] | select(.name=="apply_session_patch") | .inputSchema'
   ```

   This returns the live JSONSchema for `apply_session_patch` (or any tool you substitute in the `select(.name==...)`). Cross-reference the field list against the tool docs.

   > Note: `https://poppify.ai/.well-known/mcp.json` is a *discovery manifest* (name, endpoint, a few tool stubs) — it does NOT carry per-tool `inputSchema`, so it cannot be used for drift checks. Only `tools/list` reflects the deployed schemas.

## Drift checklist

For each tool the user asked you to verify, confirm:

- The parameter name you intended to pass is in `inputSchema.properties`
- The expected type matches (string vs enum vs boolean)
- The parameter is at the top level, not nested inside another object you forgot

## Known previously-dropped fields (now fixed)

These were silent-drop bugs that have been resolved — if you see one of them missing from the deployed schema, the deploy hasn't propagated yet:

- `apply_session_patch.textColor` — hex color for caption text (originally dropped on the old `customize` tool)
- `apply_session_patch.textPosition` — `top` / `center` / `bottom`
- `apply_session_patch.continuousEffect` — boolean, opt-out of continuous motion

If any of these are STILL missing from the live schema: the production build may be stale. File via `submit_feedback({ apiKey, frictionPoints: ["schema drift: apply_session_patch.textColor missing from deployed schema as of <date>"] })` (there is no `kind`/`description` field — drift reports go in `frictionPoints`) so the Poppify team can investigate.

## What to do if you find drift

1. **Don't keep guessing field names** — the dropped field will never land no matter how you spell it. Once you've confirmed the schema is missing the field, the only path is server-side.
2. **File feedback** with the exact field name + the tool name + the version/build of the MCP you're hitting (the MCP server emits a build ID like `poppify-00211-sl6` in its `initialize` response or error messages).
3. **Work around if urgent**: pick the closest sibling field that IS in the schema (e.g. for missing `textColor`, override the whole `textAnimation` style which may shift the color implicitly via recipe palette; not a true substitute but unblocks the user).

## Why this matters

Schema drift bugs are the most expensive bug class in MCP: the user pays seeds for a `confirm()` render, the render uses defaults instead of the requested override, and nobody knows the field was dropped until they verify the output frame-by-frame. This skill catches it BEFORE the render, not after.
