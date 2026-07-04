---
name: poppify-schedule-optimizer
description: Engagement/reach-optimized publishing via the Poppify MCP. Use whenever the user wants to POST or SCHEDULE a rendered reel and cares about performance — "post this", "schedule it", "when should I post", "best time to post", "maximize engagement", "maximize reach", "post at the best time", "optimize my posting schedule", "when does my audience engage". Covers publish_post recommendedSlots (real engagement history vs platform norms), scheduledAt:"best", slotAssessment interpretation, per-slot bestPostType, and multi-post spacing. Requires a linked wallet (wallet({action:"link"})) with channels connected in the Poppify app.
---

# Scheduling for engagement & reach — the optimizer flow

Poppify tracks engagement per `{day-of-week, hour}` bucket from the account's REAL published posts (the same data behind the mobile calendar heatmap). `publish_post` exposes it — your job is to use it instead of scheduling blind.

## Where the data comes from (so you can explain it honestly)

- `aggregates/time_slots` per user: avg engagement rate, reach, likes/comments/saves/shares, post count, `bestPostType`, and a performance tier per day×hour bucket — recomputed from published-post metrics at every sync.
- Ranking is **confidence-weighted** (`ER × log2(samples+1) × reach factor`), so one lucky post doesn't crown a slot. Two posts at 142% ER can still rank above ten posts at 2% — but say the sample size out loud when it's thin (`postCount` ≤ 3).
- **All hours are UTC.** Convert to the user's timezone when presenting, and say which timezone you're showing.

## The flow

1. After `get_result` returns `complete`, call `publish_post({apiKey, postId})` with **no channelIds**. The response carries BOTH `availableChannels` and `recommendedSlots`.
2. Check `recommendedSlots.basis`:
   - `"engagement_history"` → these are the account's own numbers. Present the top 2–3 like: *"Your audience engages best Fridays around 19:00 UTC (avg ER 4.2% across 11 posts). Next window: Fri Jul 10."*
   - `"platform_norms"` → no history yet. Say so: *"No engagement history on this account yet, so these are general platform norms (Tue–Thu midday). They'll become YOUR numbers as posts accrue metrics."* Don't oversell norms as personalized.
3. Let the user pick **channels** (never auto-select) and a **time**:
   - User says "best time" → `scheduledAt: "best"` (auto-picks the top upcoming slot; response's `pickedFrom` tells you which basis it used), or pass a specific slot's `nextOccurrence` verbatim.
   - User picks their own time → schedule it, then read `slotAssessment` in the response. If the verdict says historically weak, surface it with the better alternative — advisory, one sentence, never re-schedule without being asked.
4. "Post now" (omit `scheduledAt`) is always legitimate for time-sensitive content. For evergreen content, gently note when a top slot is < ~48h away: waiting usually beats posting into a dead zone.

## Multi-post spacing (campaigns, batches)

- Spread posts across DIFFERENT top slots — don't stack three posts into the same Friday-19:00 window; platforms suppress rapid-fire posting and the posts cannibalize each other's reach.
- Default spacing: ≥ 24h between posts to the same channel; same-day only when the user insists.
- Use slot ranks 1..N for posts 1..N rather than rank-1 for everything.

## Post-type matching

Each historical slot has a `bestPostType` (reel / carousel / static). When scheduling a reel into a slot whose history says carousels win there, that's worth a sentence — but reels are what Poppify ships, so it's context, not a blocker.

## Pre-flight gates (same as publish_post)

1. Wallet linked: `wallet({action:"link"})` — channels live on the app account.
2. Channels connected in the Poppify mobile app (OAuth is app-only). `portfolio({action:"list"})` shows what exists; switch active portfolio with `portfolio({action:"switch"})` for multi-brand users.
3. `scheduledAt` must be in the future AND before the rendered video URL expires (~7 days post-render) — the response `warnings` array flags expiry conflicts. For far-future scheduling, render closer to the date instead.

## What NOT to do

- Don't invent engagement numbers — quote only what `recommendedSlots` / `slotAssessment` return, including `postCount`.
- Don't present UTC hours as local times.
- Don't auto-pick channels, and don't silently move a user-chosen time to a "better" slot.
- Don't treat `platform_norms` as account-specific insight.
