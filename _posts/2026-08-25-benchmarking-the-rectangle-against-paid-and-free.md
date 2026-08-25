---
title: Benching the rectangle: exact inversion, BrushNet, and a paid remover walk into a warehouse
date: 2026-08-25 11:30:00 +0100
categories: [Engineering, Machine Learning]
tags: [benchmarking, diffusion, computer-vision, cost-analysis]
---

Part one ended with a locked recipe: detect the watermark three times, sample the text color, neutralize the strokes, paint one translucent rectangle at 0.94. It runs at 2.0 to 2.3 images per second and produces deterministic output.

Before committing it to several hundred thousand images, I tested it against the alternatives that kept coming up. Three challengers, in the order I tried them: exact alpha inversion (calculate the background instead of patching it), BrushNet (the free diffusion model), and watermarkremover.io (the paid service). Same three hard images every time: a band crossing a window on a flat wall, a two-layer purple badge on a dark warehouse, vertical text over wood grain.

## Challenger 1. Exact inversion, the one that should have won

The blend model says a semi-transparent watermark dims the background without destroying it. Invert the blend and the original comes back exactly:

```
background = (observed - alpha * 255) / (1 - alpha)
```

No patching, no reconstruction, no invented content. If it works, it beats everything: mathematically exact, CPU only, and the wood grain under the strokes comes back for free because it was never actually gone.

The catch is alpha. Per pixel, the true opacity varies: full value in stroke cores, partial at anti-aliased edges, zero between letters. I built per pixel alpha maps from a template extracted across sibling images, fitted the color, and ran the inversion.

Result: dark streaks and amplified noise. The division by (1 - alpha) magnifies every error in the alpha map by up to 30 times. A 0.03 alpha error becomes a visible streak at this contrast. Compression artifacts in the source JPEG became texture-shaped gouges.

Shortcoming: the math is exact but the inputs are not. Per pixel alpha cannot be recovered precisely enough from a single compressed JPEG. I dropped it. The idea stays correct in theory and the failure stays instructive in practice: an exact formula fed estimated inputs produces confidently wrong pixels.

## Challenger 2. BrushNet, the free diffusion model

BrushNet is a plug-and-play inpainting model from Tencent ARC, ECCV 2024. It turns any Stable Diffusion 1.5 checkpoint into an inpainting model with dense per pixel mask control. Free weights, runs locally, and the natural answer to "just use a model like ChatGPT does".

Getting it running took most of the effort. BrushNet is not in stock diffusers, it ships inside a pinned fork. The practical route was the IOPaint packaging, which bundles the model, the pipeline, and the monkey patches the architecture needs. One missing dependency (einops), one parameter name fix, and a trimmed encoder to turn my local inpainting checkpoint into a valid base.

Then the results, same three images, same masks, 30 denoising steps, about 9.5 seconds per image:

| image | BrushNet | the recipe |
|---|---|---|
| band over wall and window | flat tan patch, tone does not match the pale wall | panel matches the wall gradient |
| purple badge on dark warehouse | muddy brown rectangle with ghost text inside | clean white panel, text gone |
| vertical text over wood grain | flat beige slab, no grain | beige column, no grain |

BrushNet lost on all three. The reason is structural. Given a 512 pixel crop with a large rectangular mask, a diffusion model produces an average of its training data: wood-colored smear for the wardrobe, wall-colored blur for the warehouse. It synthesizes plausible texture, but plausible is not matching, and at 20 times the cost.

Shortcoming: diffusion inpainting needs a tight mask around a small object with rich surroundings. A full-width watermark band gives it neither.

## Challenger 3. The paid service

Watermarkremover.io detects marks itself and reconstructs behind them with what is almost certainly a heavier diffusion stack than anything I can run locally. It costs 0.10 to 0.20 dollars per image.

Two hard images went in.

The purple badge: they lost. Their output removed the badge and text, then filled the region with a dark smudge that reads as an editing accident. The recipe produced a clean white panel that reads as an intentional band. On the stated goal (the mark becomes a uniform rectangle), the paid service did not attempt the concept.

The vertical text over wood: they won decisively. Text gone, grain continued through the strokes so naturally that I had to diff the images to confirm it. My best attempt was close but slightly weaker on grain continuity. Every model-free approach left visible artifacts.

Scorecard: one to one.

## The pattern in the split

The two cases sit on opposite ends of a difficulty axis, and both results follow from it:

- Flat, uniform substrates (walls, badges): the recipe wins. A median patch and a constant blend are exact on flat color. Nothing is invented, nothing can be wrong.
- Structured, directional texture (wood grain): reconstruction wins. Continuing grain through erased strokes requires synthesizing texture, which is the one thing median filters and blends cannot do.

The axis is detectable at runtime. A mark found only by the rotated detection passes, sitting over a high variance region, is the signature of the hard case. The pipeline can tag it and route it.

## What the fleet run costs under each regime

The corpus is roughly 489,000 images today, 700,000 as the planning number.

| path | per image | 700K total | quality notes |
|---|---|---|---|
| the recipe, local GPU | 0 | 0 dollars, 3.5 to 4 days | wins flat cases, beige column on wood |
| BrushNet, my GPU | 0 | 0 dollars, 54 to 81 days | lost all three comparisons |
| watermarkremover.io | 0.10 to 0.20 | 70,000 to 140,000 | wins structured textures |
| FLUX Kontext or Qwen-Edit via API | 0.03 to 0.05 | 21,000 to 35,000 | hallucination risk at scale |
| self-hosted diffusion, rented GPUs | ~0.015 | 7,000 to 12,000 | needs 24 GB+ GPUs |

## The architecture the scorecard produces

Run the recipe on everything. Tag the hard signature: a mark found only by the rotated passes, over a high variance substrate. That tag routes each image one of three ways:

1. Ship it with the rectangle (the default, free).
2. Send the hard minority to a review queue (free, slower).
3. Send the hard minority to the paid API. If hard cases are 10 percent of 700K, that is roughly 10,000 dollars for the subset that actually needs reconstruction, and zero dollars for the 630,000 that do not.

The recipe is not the best tool for every image. It is the best default, and the benchmark says exactly where its limit is and what crossing that limit costs. That is all a production decision needs.
