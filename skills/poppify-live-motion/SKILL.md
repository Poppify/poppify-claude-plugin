---
name: poppify-live-motion
description: Prompting playbook for Poppify Live Motion (Veo image-to-video) — plain subject animation AND first/last-frame transitions. Use when the user wants "live motion", "animate the subject", a "before/after transformation", a "morph", "first and last frame" / "endFrameUri" interpolation, OR when a generated Veo clip looks wrong — "smoke morph", "cross-fade", "ghost double", "two of the same person", "motion goes backwards", "subject changed identity". Covers how to write the action prompt, how to generate composition-locked end frames with generate_image (reference params, consistent-frames mode), the journey overridePrompt template for bridged renders, and a failure-modes table.
---

# Live Motion prompting (Veo 3.1 image-to-video)

Two distinct modes with DIFFERENT prompting rules:

| Mode | What Veo gets | Who writes the prompt |
|---|---|---|
| **Plain i2v** (no end frame) | start image only, free-running animation | Poppify's auto prompt builder — fine as-is |
| **Bridged / interpolation** (`endFrameUri` set) | start image + last frame; Veo must land on the last frame | **YOU** — author `overridePrompt`; the auto prompt is wrong for this mode |

## 1. When to use live motion at all

- OPTIONAL per-slide upgrade (10 seeds/clip), never blanket-applied. ≤1 live slide per reel, usually the hook.
- Render the cinematic baseline FIRST, let the user review, then offer the upgrade (see `poppify-build-reel` Step 6).
- `search_live_library` before generating — cache hits are free.

## 2. Plain i2v action prompting (no end frame)

The auto prompt (5-part: cinematography + subject + action + context + style) works well here. Your only lever is `liveAction`:

- **Specific physical verbs** beat abstractions: "exhales slowly, hair drifting in the breeze" beats "looks relaxed".
- Match intensity to framing: close-ups take micro-gestures (blink, breath); wides take real movement (walks, turns).
- Respect the pairing check in the `generate_live_motion` dryRun preview (`pairing.ok` / `suggestedCamera`) — a warned-against camera layered on a big action produces warping.
- Iterate with `generate_live_motion({dryRun:true, ...})` (free, no apiKey) until the assembled prompt reads shippable.

## 3. First/last-frame transition workflow (bridged renders)

The full recipe, in order:

1. **Start image** on the slide (photo or `generate_image`).
2. **End frame via `generate_image`** with `referenceAssetId` / `referenceImageUrl` (consistent-frames mode, same cost) — MUST be composition-locked. Prompt template:
   > "Same camera position, same framing, same background/setting. Only *\<the delta\>* changes. Subject *\<geometric consequence of the action — e.g. two steps lower, closer to camera, larger in frame\>*."
3. **`update_slides({action:"set_motion_mode", slideIndex, motionMode:"live", liveAction, endFrameUri, liveDurationSeconds: 8})`** — pass 8 explicitly: the current provider (Veo 3.1 Lite) REJECTS `lastFrame` at 4s/6s with `400 INVALID_ARGUMENT ("use case not supported")`; 8 is the only working bucket for bridged renders.
4. **`generate_live_motion({dryRun:true, ...})`** (free preview) — reference only in this mode. The auto prompt is single-frame style (camera move + slow-motion suffix); do NOT ship it for a bridged render.
5. **Author the journey `overridePrompt`** (template below).
6. **`search_live_library`** (bridged renders cache in their own slot) → **`generate_live_motion({sessionId, slideIndex, overridePrompt})`**.

### The journey overridePrompt template

Describe the JOURNEY between the frames — the frames already carry the endpoints. ≤100 words, locked camera, no camera moves, no slow-motion:

> "Locked camera on *\<shot type\>* of *\<subject\>*, *\<transformation verb\>*: *\<how the change physically happens, step by step\>*, one continuous motion ending in the final-frame pose. One single continuous take: every element moves smoothly from its first-frame position to its last-frame position. Single subject throughout, consistent identity, clean clear air."

**Worked example** (a gown color transformation while descending stairs):

> "Locked camera on wide shot of a young woman on a grand staircase, descending two steps toward the camera as a wave of color sweeps down her gown from bodice to hem, the white silk turning deep emerald green like ink spreading through fabric. One single continuous take: every element moves smoothly from its first-frame position to its last-frame position. Single subject throughout, consistent identity, clean clear air, no smoke."

## 4. Transformation choreography rules

- **Mechanism, not endpoints.** "X becomes Y" makes Veo invent its own mechanism — usually a smoke/fog dissolve. Say HOW: sweeps, ripples, spreads, unfolds, settles.
- **One transformation per clip.** Locomotion + wardrobe + lighting in one 6s clip = mush. Big appearance delta ⇒ small pose/position delta (or split across two slides).
- **Geometric consistency.** If the action moves the subject, the end frame must show the spatial consequence (closer/larger, turned, relocated). An end frame that contradicts the action's geometry forces Veo to reconcile the impossible.
- A deliberate camera journey between two viewpoints (e.g. a 180° arc) is possible — describe exactly one camera path in the overridePrompt and make the end frame the destination viewpoint. Never combine a camera journey with a subject transformation.

## 5. Failure modes → cause → fix

| Symptom | Cause | Fix |
|---|---|---|
| Smoke / fog / cross-fade morph | Prompt restated endpoints, no mechanism | Rewrite overridePrompt: describe HOW the change physically happens |
| Ghost duplicate of the subject | End frame not composition-locked (camera/framing drifted) | Regenerate end frame with the lock template |
| Motion goes backwards / subject drifts away | End-frame geometry contradicts the action | Regenerate end frame showing the geometric consequence of the motion |
| Identity drift (face/outfit changes) | Too many simultaneous changes | One transformation per clip; shrink the pose delta |
| Rushed, jump-cut smear | too-short bucket on a real transformation | `liveDurationSeconds: 8` |
| `400 INVALID_ARGUMENT` ("use case not supported") on generate | `lastFrame` + 4s/6s duration on Veo 3.1 Lite | Set `liveDurationSeconds: 8` — the only bucket the provider accepts with an end frame (seeds auto-refund on the failure) |
| Clip ignores the end frame entirely (free-runs) | Provider support, not prompting | Re-check `endFrameUri` was set on the slide; if confirmed, flag it — this is a provider issue |

## 6. Duration, cost, cache

- Veo buckets: **4 / 6 / 8s** for plain i2v. **Bridged renders: 8s ONLY** — Veo 3.1 Lite rejects `lastFrame` at 4/6 (400). Explicit `liveDurationSeconds` always wins over the rounded default.
- **10 seeds/clip** (~$0.40); failures auto-refund. Cache hits via `search_live_library` are free.
- Bridged renders cache separately from plain animations of the same start image, and **override prompts get their own cache slot** — iterating on the overridePrompt never collides with earlier renders. `forceRender: true` for a deliberate re-roll.

For the wider reel flow (baseline first, upsell rules, layered camera), see `poppify-build-reel`. For verifying the finished clip/reel, see `poppify-render-debug`.
