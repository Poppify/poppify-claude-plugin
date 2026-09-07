---
name: poppify-text-card
description: Render a pixel-perfect text card (terminal frame, install command, code snippet, stat callout, title/end card, any exact literal string) as an image for a Poppify slide — locally via HTML/CSS + a headless browser, cross-platform (macOS/Linux/Windows). Use when a slide's content is the user's LITERAL words and must be crisp, because Gemini Imagen garbles typography. Try the free server-side add_text_card tool first — it needs no shell and works in every client. This local HTML/CSS + headless-browser path is for designs add_text_card cannot express (terminal frames, multi-line code). Zero seeds either way.
---

# Poppify text card (local render — a Claude Code capability)

Text-primary slides — install commands, code, terminal frames, stat callouts, exact headlines/URLs/brand names — must **never** go through `add_slide_image` (diffusion garbles literal text). On a shell-capable client you render them as a pixel-perfect HTML/CSS card and screenshot it. Zero seeds, ~30s per card.

## Try `add_text_card` FIRST — it is free and needs no shell

`add_text_card({apiKey, sessionId, slideIndex, text, ...})` renders a text card
**server-side**, costs zero seeds, and works in EVERY client — Claude Desktop,
claude.ai, Cursor, anywhere. It is the right answer for the overwhelming
majority of text-primary slides, and `add_slide_image`'s own description says
so: "For a slide whose content IS the literal string … use add_text_card
instead: exact and free."

Reach for the local render below ONLY when `add_text_card` cannot express the
design — a genuine terminal frame, multi-line syntax-highlighted code, or a
layout you need pixel control over. That is a narrow case, not the default.

## Capability gate for the local path

The local render needs a **shell** plus a **headless browser** (Chrome /
Chromium / Edge) or Playwright. This is a Claude Code capability.

- **No Bash/shell tool (Claude Desktop, web)?** Use `add_text_card` — see
  above; it has no shell requirement. Failing that, put the text in
  `update_slides({action:"set_text", slideIndex, newText})` and let the
  **composer** draw it as a caption (crisp `drawtext`, not diffusion).
- **Shell present but no browser?** Try `npx playwright screenshot` (downloads Chromium on first run), else fall back to the composer caption above.

## Step 1 — write the styled HTML (target 1080×1920 for 9:16)

```bash
mkdir -p /tmp/poppify-card
cat > /tmp/poppify-card/slide.html << 'EOF'
<!DOCTYPE html>
<html><head><meta charset="utf-8"><style>
  html,body { margin:0; padding:0; }
  body { width:1080px; height:1920px; background:#0d0518; color:#f0eef6;
         font-family:'SF Mono',Menlo,Consolas,'DejaVu Sans Mono',monospace; }
  .frame { width:1080px; height:1920px; padding:120px; box-sizing:border-box;
           display:flex; align-items:center; }
  .cmd { font-size:48px; line-height:1.4; }
  .cursor { color:#a78bfa; animation: blink 1s steps(2) infinite; }
  @keyframes blink { 50% { opacity: 0; } }
</style></head><body><div class="frame"><div class="cmd">
  $ claude mcp add poppify https://poppify.ai/mcp<span class="cursor">_</span>
</div></div></body></html>
EOF
```

Design notes: keep a dark background matching the reel's palette, generous padding (safe zones), and one idea per card. For stat callouts use a huge number + small caption; for code use a monospace block with syntax-ish coloring.

## Step 2 — HTML → PNG, cross-platform

Same flags on every OS; only the **binary name** differs. Use whichever resolves first:

```bash
# Resolve a Chrome/Chromium/Edge binary
CHROME=""
for c in google-chrome-stable google-chrome chromium chromium-browser \
         "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
         "/Applications/Chromium.app/Contents/MacOS/Chromium" \
         "/c/Program Files/Google/Chrome/Application/chrome.exe" \
         "/c/Program Files (x86)/Microsoft/Edge/Application/msedge.exe"; do
  command -v "$c" >/dev/null 2>&1 && { CHROME="$c"; break; }
  [ -x "$c" ] && { CHROME="$c"; break; }
done

if [ -n "$CHROME" ]; then
  "$CHROME" --headless=new --disable-gpu --hide-scrollbars \
    --force-device-scale-factor=1 --window-size=1080,1920 \
    --screenshot=/tmp/poppify-card/slide.png "file:///tmp/poppify-card/slide.html"
else
  # Fallback: Playwright downloads its own Chromium on first run.
  npx --yes playwright screenshot --viewport-size=1080,1920 \
    file:///tmp/poppify-card/slide.html /tmp/poppify-card/slide.png
fi

ls -l /tmp/poppify-card/slide.png   # sanity: should be a few hundred KB
```

> Windows note: in Git Bash the paths above work; in PowerShell drop the `for`-loop and call `chrome.exe`/`msedge.exe` directly with the same flags.

## Step 3 — attach to the slide

Upload the PNG and set it as the slide image. On a shell client, the presigned-PUT path is simplest:

```bash
# upload_asset({apiKey, kind:"photo", contentType:"image/png"}) → {uploadUrl, accessUrl,
#                                                                  uploadHeaders, curlExample}
# USE THE RETURNED curlExample VERBATIM. The presigned PUT binds extra headers
# (x-goog-custom-time, and x-goog-meta-filename when you pass a filename) into
# the v4 signature — a hand-written curl with only Content-Type is rejected by
# GCS with 400 MalformedSecurityHeader. Every header in `uploadHeaders` must be
# sent, exactly as given.
# then:
# update_slides({action:"set_image", slideIndex, imageUrl:<accessUrl>})
# and suppress the composer's own caption on this slide (the text is already in the image):
# update_slides({action:"set_text", slideIndex, newText:""})
```

(You can also skip the PUT and ingest server-side: `upload_asset({apiKey, kind:"photo", contentType:"image/png", dataBase64:<base64 of the PNG>})` → use the returned `accessUrl`.)

## Step 4 — cleanup

```bash
rm -rf /tmp/poppify-card
```

## Why local, not AI

Pixel-perfect literal text, zero seeds, deterministic. Any install command, code snippet, stat number, brand name, or URL that "the user must read exactly" belongs here — never in `add_slide_image`.
