---
title: How I took a 12 day image pipeline down to two days
date: 2026-08-18 09:30:00 +0100
categories: [Engineering, Machine Learning]
tags: [python, pytorch, cuda, computer-vision, benchmarking]
---

My scraper pulled 113,998 real estate listing images into MinIO. Every one carries the host site's watermark. I wanted them all clean at full resolution, and I wanted the true machine cost of that job before committing a week of GPU time to it.

The hardware is a laptop: RTX 4070 with 8 GB of VRAM, 46 GB of RAM, 20 cores. Every number in this post comes from it.

## The pipeline

Three stages. A YOLO detector (DeMark weights) finds watermark boxes. EasyOCR catches text the YOLO model was never trained on. LaMa inpainting fills the masked regions on the GPU.

The design rule I set on day one: detection is allowed to be cheap and fast, cleaning is not. Cleaning always runs on the full resolution image, so output quality never moves, no matter what trick I pull upstream. That constraint stayed true through every optimization that follows.

## Step 1. Measure before you optimize

The trap in a job like this is the urge to reach for the shiny parallel thing first. I wrote a benchmark harness instead. It generates a dataset at the listing resolution (960×1200), then runs six parallelization strategies against it:

- A: sequential, single process
- B: size grouped GPU batching
- C: two GPU processes
- D: a 16 worker CPU pool
- E and F: streaming variants where detection feeds the inpainter live

The gate is the part that makes the bench trustworthy. Every GPU variant must match the sequential reference within a mean diff of 1.0. Bit exactness is impossible because cuDNN picks different kernels for single versus batched calls, so tolerance gates it. The gate caught real bugs. A missing `no_grad` context blew batched memory up to 7 GB. A tensor slice quietly wrote 960×960 crops into a 960×1200 pipeline. Both found by the gate, both fixed before they could poison the results.

## Step 2. The first numbers

40 images, three runs, at listing resolution:

| strategy | images per second | peak VRAM | GPU util | 180K estimate |
|---|---|---|---|---|
| A sequential | 1.22 | 1.1 GB | 51% | 290 h |
| B GPU batched | 1.16 | 4.9 GB | 53% | 292 h |
| C two GPU processes | 0.37 | n/a | n/a | 738 h |
| D CPU pool, 16 workers | 0.17 | 0 | 0% | 870 h |

The honest lesson: at one megapixel, GPU batching buys nothing. LaMa's FFC layers saturate the GPU per image. The only thing batching spends is VRAM. That table killed a whole branch of ideas in one sitting.

## Step 3. Find the wall

Detection was 86% of the runtime. At four workers it cost about 5 seconds per image against 0.8 seconds of inpainting. Extrapolated to 180,000 images that is 12.1 days, eleven of them spent waiting on the detector.

The wall was detection. Everything after this is about that wall.

## Step 4. The synthetic data lied

My generated watermarks were trivially simple text. Real images from the bucket told a different story: 13 seconds per image. OCR ate 10.2 of those seconds. Craft, the text detector, crawls on real scenes with real text and real noise. YOLO needed only 0.25 seconds.

That single measurement reordered the entire strategy table. The lesson is uncomfortable and cheap: benchmark against production data, not against what you can generate in a loop.

Measured components on 40 real bucket images:

| component | CPU | GPU |
|---|---|---|
| YOLO | 0.25 s | 0.12 s |
| OCR | 10.21 s | 0.16 s |
| detection total | 13.04 s | 0.41 s |

## Step 5. Turn the detector loose on the GPU

One flag moves detection to CUDA. YOLO drops to 0.12 seconds, OCR to 0.16 seconds. A 64× turnaround on the component that was eating the job. Total detection: 0.41 seconds per image at one worker, 32× faster, with identical detections. Two workers fit in 8 GB and edge it to 0.37.

Detection stopped being the problem in an afternoon. The reason it was ever the problem is that I measured it on synthetic data first.

## Step 6. Skip the OCR that isn't needed

OCR on these images was mostly reading street signs and shop fronts. YOLO alone caught 34 of the 37 watermarked images in the sample, and three more had no watermark at all. So the pipeline skips OCR whenever YOLO confidence clears 0.7, which covers 20 of the 34 YOLO hits. OCR stays one flag away for the sources that need it.

## Step 7. Stream

The final architecture streams. One GPU detection worker feeds the GPU inpainter the moment a mask lands, no two phase job, no intermediate files. Measured 0.81 images per second end to end.

The 180,000 image job drops to 42 to 45 hours.

## Where the wall lives now

The inpainter is the wall. 41 hours at 51% GPU utilization. The GPU is idle half the time waiting on CPU side decode, prepare, and save. The next levers are FP16 and TF32 autocast, CUDA streams, and overlapping the CPU prep with GPU compute. TensorRT only gets a look after that.

The progression, one table:

| version | per image | 180,000 images |
|---|---|---|
| baseline, CPU detection | 13.5 s | 12.1 days |
| sixteen detection workers | 5 s | 5 days |
| GPU detection | 1.2 s | 2.5 days |
| GPU detection plus streaming | 0.85 s | 1.8 days |

## Reproduce it

```
python3 -m venv .venv
.venv/bin/pip install torch ultralytics easyocr simple-lama-inpainting opencv-python pillow numpy
export LAMA_MODEL=/absolute/path/to/big-lama.pt
export WATERMARK_MODELS_DIR=/absolute/path/to/models
.venv/bin/python tests/bench_parallel.py --res 960x1200 --n 40
.venv/bin/python tests/calibrate_real.py <image_folder> 0.45 0.4 cuda 1 0.7
```

Point the env vars at wherever you keep the model weights so nothing downloads at runtime. The calibration script reads real images from any folder and reports the YOLO versus OCR split, the confidence distribution, and the skip OCR savings per threshold. Download a sample of production images and feed them in, that is the whole point.

## The strategic core

Measure the real pipeline against real data before optimizing. Every wrong turn here, the batching that bought nothing, the OCR that hid in plain sight, the synthetic set that flattered the numbers, came from skipping that step.

The tooling that made it survivable was the gate, the bench, and the discipline to let a 12 day number sit in the table until it was true. The numbers moved because I kept moving the wall.
