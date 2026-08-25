---
title: Seven attempts to replace a watermark with a rectangle
date: 2026-08-25 09:00:00 +0100
categories: [Engineering, Machine Learning]
tags: [python, opencv, computer-vision, watermark, image-processing]
---

> This story got a deeper rewrite as a four part series with diagrams and code. It starts at [How do we remove watermarks](/posts/how-do-we-remove-watermarks/).

This is part one of two about cleaning watermarks off scraped real estate photos. The previous post ended with a LaMa inpainting pipeline that erased watermarks by reconstructing what was behind them. This post is what happened when I changed the goal: instead of erasing the mark, replace it with one uniform translucent rectangle that carries the mark's own color and opacity. The text becomes unreadable, the photo keeps a band that looks like a deliberate watermark, and nothing is invented.

The constraint I set before starting: no generative model touches the output. Everything must be simple image operations whose result I can predict before running them.

What follows is the actual sequence of attempts, in order. Each one exists because the previous one failed in a specific, measurable way.

## Attempt 1. The opaque sticker

The simplest thing that could work. Detect the watermark, take the median color of every pixel under the detection mask, fill the bounding box with that color at full opacity.

The median is the right sample because the mask covers letters and the gaps between them. Mixing both gives the blended appearance of the mark rather than the pure text color.

Result: the text disappeared completely. The cost was opacity. The rectangle read as a sticker pasted on the photo. A real semi-transparent watermark lets the surface show through; a solid fill does not. On a brown floor the panel was a flat tan block. On anything busy it dominated the frame.

Shortcoming: zero transparency. The next attempt had to add it without giving up the simplicity.

## Attempt 2. Measured transparency

The blend model makes transparency estimable. A semi-transparent white mark lifts every pixel it touches toward 255, and the lift divided by the headroom to white is the opacity:

```
alpha = (observed - background) / (255 - background)
```

I ran the detection, computed this ratio over the masked pixels, and rendered the rectangle at the measured opacity instead of full. Then a ladder test: the same image at ten opacity levels from 0.40 to 0.85, then eight more from 0.86 to 1.00, each labeled, side by side.

Result: the measured opacity clustered between 0.26 and 0.31 across 40 real images, which matched the visual faintness of the original marks. At 0.94 the text was completely gone and the panel read as an intentional watermark band. I locked 0.94.

Shortcomings appeared when the batch hit the wider corpus. Two images broke it, each in a different way.

Failure one: a two-layer watermark. A solid purple badge with white text on it, over a dark warehouse. The median sampled the badge color because the badge covers more pixels than the letters. The rectangle came out purple, and the white letters ghosted through it, because painting the text's own background lighter does not hide text that is already brighter than everything around it.

Failure two: a vertical watermark running down a wardrobe. The detector caught roughly half of it, so the rectangle covered half the text. The rest stayed readable.

Both failures were diagnosable from the output alone, which is the one good property of a recipe this simple.

## Attempt 3. Sample the text, not the badge

The badge failure fixed itself inside the blend model. The text color and the badge color differ in brightness, and the text is the brighter set. Taking the median of only the brightest 20 percent of mask pixels selects the glyph strokes and ignores the badge body.

Result: on the badge image the sampled color went from purple to near white. The panel no longer fought the text.

Shortcoming: this alone did not remove the ghost. Sampling a better color still paints over pixels that contain the strokes. White veil over white strokes is a no-op.

## Attempt 4. Neutralize before painting

If painting over strokes cannot hide them, the strokes must be gone before the paint. A median filter does this: replace every glyph pixel with the median of its 21 pixel neighborhood. On a flat wall the median is the wall. On wood the median is the local grain tone. Two guards keep it from eating the photo: only replace pixels where the median differs from the original by more than 15 luminance levels, and skip near saturated surfaces where the veil alone already hides everything.

Result: the ghost disappeared on both failure images.

Shortcoming: none on its own, but the vertical detection failure was still open, and a new question had appeared. If the mark is semi-transparent, the original content is still inside every stroke pixel, dimmed but present. Could the recipe skip patching and recover the background exactly instead?

## Attempt 5. Exact inversion, the elegant dead end

The blend equation inverts cleanly:

```
background = (observed - alpha * 255) / (1 - alpha)
```

With a per pixel alpha map, this recovers the true background under every stroke, grain and all, with no reconstruction at all. I built per pixel alpha maps from a template extracted across sibling images and ran the inversion.

Result: dark streaks and amplified noise. The template carried compression artifacts and a few pixels of misalignment, and the inversion divides by (1 - alpha), which amplifies every error in alpha by up to 30 times. A 0.03 error in a 0.28 alpha becomes a visible streak.

Shortcoming: the math is exact but the inputs are not. Per pixel alpha cannot be known precisely enough from a single JPEG. The idea is correct in theory and unusable in practice at this image quality. I dropped it and went back to the median patch.

## The recipe, locked

With the badge and sampling fixed, the remaining failure was detection coverage. The final recipe adds the rotation trick and locks every parameter:

1. Detect three times per image: YOLO at 0.45 with faint OCR on the original, plus YOLO on the image rotated 90 degrees each way. Map the masks back and union them. This catches vertical marks the plain detector halves.
2. Dilate the union by 5 pixels for the bounding box, padded 8 more. Keep the raw mask for the next two steps.
3. Sample the text color as the median of the brightest 20 percent of raw mask pixels.
4. Neutralize glyph pixels with the guarded median patch.
5. Paint one constant rectangle at alpha 0.94.

Every step exists because a specific failure demanded it. The rotations exist because of the wardrobe. The brightest 20 percent exists because of the purple badge. The median patch exists because of the ghost. The 0.94 exists because of the ladder.

## Validation

The locked recipe ran on 40 real images at 2.0 to 2.3 images per second and the output was compared pixel by pixel against the batch that produced it. Maximum mean difference: 0.81, which is JPEG encoder rounding between two encoders at different quality settings. Zero images differed beyond that.

Then the same three hard images went through a free diffusion inpainting model and a paid removal service. That comparison is part two.
