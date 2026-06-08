---
name: poppify-schema-introspect
description: Inspect the deployed Poppify MCP tool schemas to verify what parameters are actually accepted, vs what older docs may claim. Use when a parameter you passed appears to have been silently dropped, OR before recommending a parameter you're not 100% sure is in the current build.
---

# Inspecting the deployed Poppify MCP schemas

MCP clients strip undeclared fields before forwarding to the server. If a parameter you pass to a tool seems to have been ignored, the most likely cause is **schema/code drift** — the server's handler accepts the field, but the MCP tool registration's Zod schema doesn't declare it, so the field was dropped at the protocol boundary.

This skill teaches Claude how to verify the deployed schema directly.

## Where to look

The MCP server publishes its tool catalog via the standard MCP `tools/list` request. Claude Code already calls this on connect — the tool definitions are available in the client's MCP state.

To inspect what was returned:

1. List the tools Claude Code currently sees:
   ```
   /mcp poppify
   ```
   That shows the names, but not the param shapes.

2. For the param shapes, look at the deployed schema directly. The canonical source-of-truth lives at:
   - **MCP endpoint**: `https://poppify.ai/api/mcp` (POST, JSON-RPC)
   - **Listing JSON**: `https://poppify.ai/.well-known/mcp.json`

   Fetch the well-known doc:
   ```bash
   curl -s https://poppify.ai/.well-known/mcp.json | jq '.tools[] | select(.name=="customize") | .inputSchema'
   ```

   This returns the live JSONSchema for `customize` (or any tool you substitute in the `select(.name==...)`). Cross-reference the field list against the tool docs.

## Drift checklist

For each tool the user asked you to verify, confirm:

- The parameter name you intended to pass is in `inputSchema.properties`
- The expected type matches (string vs enum vs boolean)
- The parameter is at the top level, not nested inside another object you forgot

## Known previously-dropped fields (now fixed)

These were silent-drop bugs that have been resolved — if you see one of them missing from the deployed schema, the deploy hasn't propagated yet:

- `customize.textColor` — hex color for caption text
- `customize.textPosition` — `top` / `center` / `bottom`
- `customize.continuousEffect` — boolean, opt-out of continuous motion

If any of these are STILL missing from the live schema: the production build may be stale. File via `submit_feedback({ kind: "schema_drift", description: "customize.textColor missing from deployed schema as of <date>" })` so the Poppify team can investigate.

## What to do if you find drift

1. **Don't keep guessing field names** — the dropped field will never land no matter how you spell it. Once you've confirmed the schema is missing the field, the only path is server-side.
2. **File feedback** with the exact field name + the tool name + the version/build of the MCP you're hitting (the MCP server emits a build ID like `poppify-00211-sl6` in its `initialize` response or error messages).
3. **Work around if urgent**: pick the closest sibling field that IS in the schema (e.g. for missing `textColor`, override the whole `textAnimation` style which may shift the color implicitly via recipe palette; not a true substitute but unblocks the user).

## Why this matters

Schema drift bugs are the most expensive bug class in MCP: the user pays seeds for a `confirm()` render, the render uses defaults instead of the requested override, and nobody knows the field was dropped until they verify the output frame-by-frame. This skill catches it BEFORE the render, not after.
