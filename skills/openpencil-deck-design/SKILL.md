---
name: openpencil-deck-design
description: Use when building or revising multi-page OpenPencil decks with images, overlays, and Chinese text.
tags: ["自习"]
---

# OpenPencil Deck Design

Build and verify multi-page OpenPencil presentations by controlling layer order explicitly, styling text with supported paint arrays, and checking every page by screenshot. This skill covers composition and visual QA; it does not generate authoritative product claims or replace source verification.

## When to Use

Use this skill when the user asks to create, rebuild, or visually refine a multi-page OpenPencil presentation, especially when it contains remote images, Chinese copy, gradients, or layered compositions.

Do not use it for:
- One-off image editing that does not involve an OpenPencil document.
- Publishing a deck without checking the rendered pages.
- Treating visual success as factual verification of product names, specifications, or availability.
- Inventing a document schema or relying on a tool call that was not observed in the session.

## Procedure

1. Establish the page skeleton.
   - Create one OpenPencil Page per slide, with exactly one top-level slide frame on that Page. Do not place all slide frames on one Page and rely on their shared `x=0,y=0` position to separate them; that creates apparent overlap.
   - Give each slide frame an explicit canvas size and `layout":"none"` when elements are positioned manually.
   - Confirm page separation with `mcp_openpencil_list_pages {}` and `mcp_openpencil_get_document_info {}`. Then call `mcp_openpencil_read_nodes` with `depth:0` and each Page ID to verify that each Page returns one 1920×1080 frame at `x:0.0,y:0.0`.
   - The session used 1920×1080 frames named `01-封面`, `02-阵容`, `03-Pro相机`, `04-性能`, `05-Duo折叠`, `06-续航`, `07-设计`, and `08-总结`.
   - Use `mcp_openpencil_batch_design` operations of the form `p1=I(null,{"type":"frame","name":"01-封面","width":1920,"height":1080,"x":0,"y":0,"layout":"none","clipContent":true,"fill":[{"type":"solid","color":"#0A0A0C"}]})`.

2. Confirm layer order before building the full deck.
   - Insert a small disposable overlap probe with two differently colored rectangles.
   - The observed and verified rule is: the child inserted first is rendered on top. `children[0]` is topmost.
   - Insert foreground elements first and background elements last. For a normal slide, use the order: text and foreground overlays → panels/scrims → images → background.
   - A proven frame-layer pattern is:
     - `tl=I(zz,{"type":"frame","name":"textLayer","x":0,"y":0,"width":300,"height":200,"layout":"none"})`
     - `tt=I(tl,{"type":"text","name":"TXT","x":20,"y":20,"width":260,"content":"HELLO","fontSize":40,"fontFamily":"Inter","fill":[{"type":"solid","color":"#ffffff"}]})`
     - Insert `tl` before a background layer so the text layer remains above it.
   - Delete disposable probes with `D("n18")` after the rule is confirmed.

3. Create text with explicit paint and font settings.
   - Use `content` for the copy, a sufficient `width`, and `fill` as an array containing a solid paint.
   - A working Chinese text form is `I("n114",{"type":"text","name":"T1","x":20,"y":20,"width":300,"content":"可变光圈 折叠","fontSize":40,"fontFamily":"Noto Serif SC","fill":[{"type":"solid","color":"#ffffff"}]})`.
   - Multiline Chinese content was confirmed to render in a text node with an explicit width.
   - Do not infer text color from a screenshot when the node has an empty `fill_ref`; verify the actual rendered result and update the node's fill explicitly if needed.

4. Add images with known dimensions and safe fit behavior.
   - Use an image node with explicit `x`, `y`, `width`, `height`, `src`, and `objectFit":"fit"`.
   - Example: `im=I("n1",{"type":"image","name":"cover-img","x":760,"y":0,"width":1160,"height":1080,"src":"https://www.apple.com.cn/v/iphone-18-pro/c/images/overview/media-hero/hero_endframe__fbxsnz8txcia_large.jpg","objectFit":"fit"})`.
   - Remote image URLs were confirmed to load, and `objectFit":"fit"` avoided stretching. `objectFit":"crop"` was not reliably distinguishable in the probe.
   - Before placement, inspect local image dimensions and corner background colors with a short Python/Pillow script. Preserve the source aspect ratio and choose dark or light slide treatments according to the image background.
   - For remote images, preflight every URL for HTTP status, content type, and a nonzero body size before insertion. A `200` response alone is not sufficient: the rendered image must also be checked for a gray or broken-image placeholder.
   - If a remote URL returns `404 未找到`, replace it with a verified asset URL and reinsert the image node with explicit `src`, `objectFit`, `x`, `y`, `width`, and `height`; then verify the rendered node rather than only the write response.
   - For white-background product assets on dark slides, place them inside a deliberate light panel rather than assuming a transparent cutout.

5. Apply overlays without changing the intended hierarchy.
   - Use a full-slide or partial linear-gradient rectangle for a scrim, with stops that transition from opaque to transparent.
   - Keep the scrim below all foreground text. Because the first inserted child is topmost, insert the scrim before the text.
   - Avoid changing a node's fill with `U` when doing so may alter its position in the child order. If hierarchy becomes uncertain, recreate the affected nodes or explicitly move them with `M`; the observed move form was `M("n10","n1",1)`.
   - After a move or update, take a screenshot rather than assuming the visual result.

6. Build slides in coherent batches.
   - Use one `mcp_openpencil_batch_design` call for related insertions, but keep the insertion order explicit within the operation string.
   - Use bindings such as `im`, `sc`, `t1`, and `t2`, then use the returned `nodeId` values for follow-up updates, moves, screenshots, or deletion.
   - Separate operation groups for independent pages so a single malformed node does not obscure the page under review.

7. Verify each page visually.
   - Call `mcp_openpencil_get_screenshot` with the page's `nodeId`, for example `mcp_openpencil_get_screenshot {"nodeId":"n1"}`.
   - Inspect image loading, text color, Chinese glyphs, gradient boundaries, clipping, alignment, and whether overlays obscure foreground text.
   - Use `mcp_openpencil_get_node` when a screenshot is ambiguous, for example `mcp_openpencil_get_node {"node_id":"n13"}`.
   - Repeat the screenshot after each corrective update; do not treat a successful write response as visual verification.

8. Verify factual content separately from rendering.
   - For current product names or availability, query the authoritative site rather than inferring from the deck's subject.
   - In this session, `Invoke-WebRequest -UseBasicParsing 'https://www.apple.com.cn/shop/buy-iphone'` was used to inspect the current purchase list.
   - Keep factual verification distinct from OpenPencil visual verification.

## Pitfalls

- A successful `mcp_openpencil_batch_design` response is not proof that the page looks correct; inspect the rendered screenshot.
- Later insertion is not the top layer. In OpenPencil, the child inserted first is on top, so inserting text last can put it beneath an overlay.
- `U` can make a previously safe hierarchy suspect when changing a node's fill; re-screenshot or explicitly reorder it.
- Text without an explicit `fill` array may render ambiguously. Set the solid fill and font explicitly, then verify glyphs.
- `objectFit":"crop"` was not reliably distinguishable from `fit` in the probe, so use `fit` unless crop behavior is independently confirmed.
- A remote URL in an image node can still fail to load, leave a blank region, or place a white-background asset poorly on a dark slide.
- Eight frames with the same `x=0,y=0` on one Page can look like one overlapping stack. Verify one Page per slide, not merely one frame per slide.
- HTTP `200` proves the asset was fetched, not that OpenPencil rendered it. Check for a visible placeholder after insertion.
- A stale CDN URL can return `404 未找到`; validate URLs at insertion time and again after corrections.
- A full-slide scrim with opaque stops can obscure all copy even when the intended text node exists.
- A screenshot that shows the expected text shape does not establish that its color, hierarchy, or factual content is correct.
- Do not claim a deck is complete merely because the frame count equals the requested page count; every page needs a visual check.

## Verification

Confirm that `mcp_openpencil_list_pages {}` shows one Page per requested slide and that `mcp_openpencil_read_nodes` at `depth:0` shows exactly one 1920×1080 frame on each Page. Confirm that `mcp_openpencil_get_screenshot` renders the intended image and readable text for every Page, with foreground text above gradients and panels. The final check must show the expected number of visible Pages with no probe frames, blank images, clipped copy, overlay-obscured headings, or broken-image placeholders.
