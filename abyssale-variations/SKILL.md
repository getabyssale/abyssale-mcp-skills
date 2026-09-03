---
name: abyssale-variations
description: Generate multiple format/size/language variations or a video from an existing Abyssale design in one go. Use when the user asks to "generate N variations of this banner", "make this in every format", "turn this into a video", or wants several creative outputs derived from one Abyssale design. Not for multi-page PDFs from a printer_multipage design — see the abyssale-print-catalog skill for that.
---

# Generate variations from an Abyssale design

Abyssale can turn one design into many rendered outputs — different formats, languages, or a video — without recreating the design each time. This skill picks the right generation call and follows through to finished assets.

Note: `printer_multipage` designs are a different design type and are out of scope here — `generate_static_banner`/`generate_banners_async` both reject them. Use the `abyssale-print-catalog` skill for those.

## Prerequisites

The Abyssale MCP server must be connected with the `generate:banner` scope (and `read:design` if you need to find the design first).

## Steps

1. **Identify the design.** If the user names a design, use `search_designs` or `list_designs` to find its identifier, then `get_design` to confirm it's the right one (check dimensions/layers match what they expect) before generating from it.

2. **Choose sync vs async:**
   - A **single** static image needed right away → `generate_static_banner`. Synchronous, capped at 10 seconds server-side, returns the result directly. Good for one quick preview.
   - **Multiple** formats/sizes, a **video**, or anything AI-assisted → `generate_banners_async` instead. Use this even when it "looks like" just one more image on top of others you're already generating — batching variations through the async path is more reliable than looping the sync one, and it's required for anything that can't render in 10 seconds.

3. **Submit the generation(s).** For N variations (N formats, N languages, N text swaps, …), call the async tool once per variation with its specific overrides — each call returns a `generation_request_id` immediately. Submit all of them before waiting on any; don't wait for one to finish before submitting the next.

4. **Poll for completion.** Call `check_generation_status` per `generation_request_id`:
   - Finalized → the asset is ready; collect its URL.
   - Still pending → wait briefly and poll again — this is normal, not an error.
   - Expired (after 7 days) → if the user asks about an old request much later, tell them it expired rather than retrying the same id.

5. **Report results as a set**, once every requested variation is finalized — the user asked for N variations, so the useful unit of output is the whole batch, not one-at-a-time updates.

## Notes

- Don't fake multi-format generation by looping `generate_static_banner` — it's built for a single fast render, and its 10-second cap will fail on anything heavier.
- If the request implies a video, don't try to force it through the static/sync path — it doesn't support that.
