# RFC: Studio as the creative suite — every content type, one flywheel

- **Start Date**: 2026-07-18
- **Author(s)**: @zooqueen
- **Status**: Draft
- **Related**: studio-agentic-creative-runtime.md

## Summary

Studio today ships one content lane — image — end to end: generate from CAD or
reference, organize in Stacks, fix with lineage, rate with feedback that
conditions the next render, quality-gate every output, render on your own GPU
or ours. This RFC makes that lane the TEMPLATE for every creative content type
— video, music, decks, documents — so Studio becomes the suite: Canva's breadth
with a generation flywheel Canva does not have. One library, one queue, one
feedback loop, one wallet; a lane per medium.

## Motivation

Teams do not want five tools with five libraries. The differentiators Studio
already has — provenance per job (graph, seed, params), the rating→conditioning
flywheel, bring-your-own GPU — apply to every medium unchanged. The cost of a
new medium is a lane, not a product.

## Design

### The lane contract

A content type is ONE lane implementing four seams; everything else is shared.

| Seam | What it is | Image lane (shipped) |
|---|---|---|
| generate | graph templates the composer + templates grid dispatch | qwen-image-edit family |
| card | how the library renders it (thumb, player, badges) | `.card` thumb |
| edit | the surface "Fix"/"Open in Editor" opens | fix dialog + node editor |
| judge | what the quality gate asks the vision/audio model | photoreal skin/design gate |

Shared, untouched by new lanes: library + Stacks + Archive, worklog identity
(unique per-job prefixes), queue + history + lineage, ratings → do-not-repeat
conditioning + feedback.jsonl, wallet + GPU fleet, org tenancy, analytics.

### Lane order and reality

1. **Video** — ✅ SHIPPED (2026-07-18). The Wan 2.2 TI2V-5B lane over the ONE
   library/queue/flywheel: `_video_graph` (proven node-for-node on spark, ~154s
   for 640×640×3s), `POST /v1/library/video` (text-to-video or one reference
   image), mp4/webm ingest + `<video>` poster cards, a motion quality gate
   (start/mid/end frames), and the Video template card. Built → adversarially
   verified (composer-lane-reset blocker fixed) → merged. No new infra.
1b. **3D** — ✅ SHIPPED (2026-07-18). Hunyuan3D-2.1 image-to-mesh on the GB10:
   `_model3d_graph` (ImageOnlyCheckpointLoader bundled model+CLIP+VAE +
   ModelSamplingAuraFlow — both non-obvious and load-bearing) → `.glb`,
   `POST /v1/library/model3d` (one reference image, no text-to-3D), glTF ingest,
   and an interactive `<model-viewer>` zoom card. Verified: a clean 3D garment
   mesh from a ghost-mannequin shot. No new infra.
2. **Music / audio** — engine-served (ACE-class TTA model on the fleet); audio
   cards with waveform + play; gate = audio model scoring. Renders are cheap;
   the flywheel (notes like "too much reverb") ports verbatim.
3. **Decks** — different shape: a deck is a COMPOSITION of library assets +
   text. Editor surface = HTML deck editor (one slide = one canvas, assets
   dragged from the library rail); generation = per-slide image lanes plus a
   copilot outline pass. Export: PDF/PNG via the render fleet. This is where
   "Canva breadth" lands — and every slide image is already flywheel-backed.
4. **Documents / brand kits** — templates + design-system chips already in the
   composer become enforceable: a brand kit is a Stack of approved assets +
   palette + type rules the copilot injects into every lane's conditioning.

### Editor architecture

One editor shell (the existing Studio⇄Editor toggle), N surfaces. A surface
registers `{contentType, mount(el, asset), save() -> asset}`. The node editor
is the image surface; video gets trim/poster controls on the same shell; decks
get the slide canvas. No per-medium apps — the shell, chrome, chat, wallet,
and library rail are one implementation.

### What we explicitly do NOT build

- No separate video/music/deck libraries, queues, or billing.
- No third-party embed editors — surfaces are ours, small, and composable.
- No lane ships without its judge: ungated generation is how catalogs rot.

## Rollout

Video lane first (it proves the lane contract with zero new infra), then audio,
then the deck surface. Each lane lands behind its template card — no flags, no
env gates; a lane is either shipped whole or absent.

## Open questions

- Deck export fidelity target (PDF print vs web share) — decides the renderer.
- Music licensing posture for generated audio in client work.
- Whether the copilot outline pass for decks runs on the org's zen router or
  the default session model.
