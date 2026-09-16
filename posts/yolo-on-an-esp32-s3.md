---
title: Running YOLOv5n on an ESP32-S3 — and when not to
date: 2026-03-14
description: For an IEEE CIS robot prototype I got a quantised YOLOv5n running entirely on an ESP32-S3. It works. It also takes 3–5 seconds a frame, which taught me more about edge AI than the part that worked.
tags: [embedded, ai, esp32, tflite]
---

For an autonomous vehicle prototype with the IEEE Computational Intelligence Society chapter, I wanted object detection to run on the robot itself — no laptop in the loop. The board was an ESP32-S3; the model, YOLOv5n, the smallest of the YOLOv5 family.

This is how that went, the numbers it came out at, and the architectural decision those numbers forced.

## The conversion pipeline

A PyTorch model cannot run on a microcontroller directly, so it goes through three conversions:

```text
yolov5n.pt  ──►  ONNX  ──►  TFLite (full-integer quantised)  ──►  model.h
 (PyTorch)      (export)        (calibrated to uint8)            (C array)
```

Each step is its own small script: export to ONNX, convert to TFLite with integer quantisation, and dump the `.tflite` bytes into a C header the firmware can compile in.

The model that comes out the other end:

- **Input:** 96×96 RGB, uint8
- **Output:** `[1, 567, 85]` — 567 candidate boxes, each with 4 box coordinates, 1 objectness score and 80 COCO class probabilities
- **Quantisation:** full integer, so every value is a uint8 plus a shared scale (≈0.01026) and zero point (0)
- **Size:** ~12.7 MB as a C header

96×96 is tiny for detection. It was the resolution that fit, not the one I wanted — and it caps what the model can see before any other limitation kicks in.

## Fitting it in memory

The memory budget is the whole game on a microcontroller:

```text
ESP32-S3
├── Flash (16 MB)
│   ├── model.h .............. ~12.7 MB
│   └── sketch ............... ~0.5 MB
├── SRAM (512 KB)
│   ├── tensor arena ......... 200 KB
│   ├── stack + globals ...... ~100 KB
│   └── Wi-Fi/TCP buffers .... ~50 KB
└── PSRAM (8 MB) ............. image buffers
```

This is why the S3 specifically: the N16R8 variants have 16 MB of flash for the model and 8 MB of PSRAM for image buffers. A standard ESP32 simply does not have the room. TensorFlow Lite Micro (via the Chirale library for Arduino) runs the model inside a fixed 200 KB tensor arena allocated once at startup.

Two failures I hit that are worth knowing about:

- **"Guru Meditation Error"** on boot is usually a stack overflow, and on this board it usually means PSRAM is not enabled in the board settings.
- **All detections are garbage** almost always means the input is wrong — BGR instead of RGB, or floats instead of uint8. The model will not complain; it will just be confidently wrong.

## Reading the output

Every value in the output tensor is quantised, so each one has to be scaled back before it means anything. Parsing is a loop over the 567 boxes: dequantise the objectness score, skip the box if it is under the confidence threshold (0.5), otherwise dequantise the box and take the argmax over the 80 class scores.

```cpp
float real = (uint8_value - ZERO_POINT) * SCALE;
```

## Feeding it images

I tried several image sources, and the robot firmware ended up with a few of them:

- **HTTP over Wi-Fi** — the board fetches a frame from a small image server
- **Serial over USB** — frames pushed from a PC, no network needed
- **IP camera** — a JPEG stream decoded on the board

## The number that mattered: 3–5 seconds

Here is how the approaches compared:

| Approach | Where inference runs | Latency per frame |
| --- | --- | --- |
| On-device, image over HTTP | ESP32-S3 | ~3–5 s |
| On-device, image over serial | ESP32-S3 | ~3–5 s |
| On-device, IP camera (JPEG decode) | ESP32-S3 | ~5–8 s |
| Offloaded to a PC | PC | ~30–100 ms |

On-device detection works — the robot really does detect objects with nothing but the microcontroller. But a vehicle that looks at the world every four seconds is not driving on what it sees. And inference blocks: during those seconds the chip is not servicing Wi-Fi, so connections drop unless everything around inference is written non-blocking.

That is a gap of one to two orders of magnitude, and no amount of tuning a 96×96 YOLO on a microcontroller closes it.

## The split that actually works

So the architecture that makes sense for a moving robot is a split:

- **The ESP32 does what microcontrollers are good at:** motor control, reading sensors, and navigation.
- **Vision runs wherever latency allows.** For driving, that means offloading frames to a nearby machine. For slower, standalone or privacy-sensitive jobs — "is there a person in this room?" — on-device inference is genuinely useful, because no image ever leaves the board.

The navigation side of the robot firmware runs a D\* Lite–style planner on the ESP32 itself: a grid of cells with `g` and `rhs` cost estimates and a priority queue keyed on `min(g, rhs) + heuristic`. The drive loop steers toward the next cell on the path by driving the angle error between the robot's heading and that target to zero.

Detection and planning meet in one small function. A detection that is large and near the centre of the frame is treated as something in the way: it is projected to a point 30 cm ahead along the robot's current heading and marked as an obstacle cell. D\* Lite then updates only the affected cells and their neighbours instead of replanning from scratch, which is exactly why it suits a robot that discovers its map as it goes. Unlike detection, that loop is cheap enough to run continuously on the chip.

The fixed 30 cm is the honest weak spot: a single camera frame carries no depth, so the distance is an assumption. A cheap ultrasonic or ToF sensor to measure it would make the obstacle map far more trustworthy.

## What I'd take away

- **Measure latency first.** "It runs on the device" and "it is fast enough for the device's job" are different claims, and only the second one matters.
- **Design for the memory map, not the model.** The model was chosen by what fits in flash and a 200 KB arena, and that is the right way round on a microcontroller.
- **Look at purpose-built runtimes.** Espressif's ESP-DL is optimised for the S3's vector instructions and is the obvious next thing to try before giving up on on-device detection entirely.
