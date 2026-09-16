---
title: Teaching a glove to recognise gestures on a microcontroller
date: 2025-08-24
description: Cogni-Glove started as an air mouse and grew a gesture classifier — a tiny 1D CNN, quantised to int8, running under TensorFlow Lite Micro next to the sensor.
tags: [embedded, ai, tflite, esp32]
---

Cogni-Glove is a wearable input device: an MPU-6050 accelerometer/gyroscope and a hall-effect sensor on a glove, wired to a microcontroller. It began as a way to move a mouse cursor by tilting my hand. The second version adds on-device gesture recognition, which is the more interesting half and most of this post.

## Version 1: an air mouse in 60 lines

The first version needed no machine learning at all. The board streams raw sensor readings over serial as tab-separated `KEY:value` pairs:

```text
AX:123	AY:456	AZ:789	GX:101	GY:112	GZ:131	HALL:1
```

A Python script on the PC (`ser_mouse.py`) turns two gyro axes into cursor motion and the hall sensor into a click. Three small tricks made it usable rather than twitchy:

- **Deadzone** — gyro values below 800 in magnitude are treated as zero, so a resting hand does not drift the cursor.
- **Moving average** over the last 3 readings, to smooth sensor jitter.
- **Sensitivity scaling** (×0.003) to map angular rate to pixels.

It works, and it also makes the limitation obvious: tilt-to-move is continuous control. Anything discrete — "that was a swipe", "that was a flick" — needs to recognise a *shape* in time, not a single reading.

## Collecting data

Gestures are learned from examples, so the first tool is a recorder. `rec_gui.py` is a small Tkinter app that connects to the board over serial, plots the live signal with matplotlib, and records labelled windows into `recordings.csv`. Label names are mapped to integer targets in `strToTarget.json`, so adding a gesture is a matter of typing a new name and recording.

Each row is one example: a **1-second window at 128 Hz** — 128 samples × 6 features (accel X/Y/Z, gyro X/Y/Z) flattened into 768 numbers — followed by its label. `combine_csvs.py` merges sessions recorded separately.

## The model is deliberately small

The model has to fit in a microcontroller's RAM, so it is a compact 1D convolutional network over the time axis:

```python
inp = tf.keras.Input(shape=(128, 6))
x = tf.keras.layers.Conv1D(16, 5, activation='relu', padding='same')(inp)
x = tf.keras.layers.Conv1D(16, 3, activation='relu', padding='same')(x)
x = tf.keras.layers.AveragePooling1D(pool_size=2)(x)
x = tf.keras.layers.Conv1D(32, 3, activation='relu', padding='same')(x)
x = tf.keras.layers.GlobalAveragePooling1D()(x)
x = tf.keras.layers.Dense(32, activation='relu')(x)
out = tf.keras.layers.Dense(num_classes, activation='softmax')(x)
```

1D convolutions are a natural fit for IMU data: a gesture is a pattern that can happen anywhere in the window, and convolution plus global pooling finds it regardless of exactly when it started.

Training (`train.py`) normalises each feature by its mean and standard deviation, holds out 15% as a stratified validation split, and trains for 40 epochs with Adam.

## Quantising for the chip

A float model is the wrong thing to ship to a microcontroller. The training script converts it to TensorFlow Lite with **full integer quantisation**: int8 weights, int8 activations, int8 input and output.

Integer quantisation needs to know the real range of every activation, so the converter is given a representative dataset — the first 100 training windows — to calibrate the scales and zero points:

```python
converter = tf.lite.TFLiteConverter.from_keras_model(model)
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_data_gen
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.int8
converter.inference_output_type = tf.int8
```

The `.tflite` file is then embedded into the firmware as a C array (`model.h`).

## Running it next to the sensor

The firmware (`main_infer.ino`) uses TensorFlow Lite Micro through the Chirale library. The interpreter runs inside a static **60 KB tensor arena** — all of the model's working memory, allocated once at boot.

The loop is straightforward:

1. Read a 6-axis frame from the MPU-6050 over I²C and scale it (accelerometer to g, gyro to °/s).
2. Normalise with the training mean and std, and append to a float buffer.
3. When 128 samples have accumulated, quantise the buffer into the int8 input tensor using the tensor's scale and zero point.
4. `Invoke()`, dequantise the output scores, take the argmax, and print the class and its score.

### The bug that is easy to ship

Training saves the normalisation constants to `model_metadata.json`. The firmware has its own copies, `MEAN[6]` and `STD[6]`, which must be updated by hand after every retrain. If they are left at their defaults (0 and 1), nothing crashes — the model just sees inputs on a completely different scale than it was trained on and quietly predicts nonsense.

That is the classic train/serve skew, in miniature. The fix I want is to generate those two arrays straight into a header during conversion, the same way the model weights already are, so they cannot drift apart.

## What I'd do next

- **Sliding windows.** The buffer resets after each inference, so the firmware makes one prediction per second and a gesture that straddles two windows can be missed. A ring buffer with a half-window hop would roughly halve the latency for the same model.
- **A confidence floor and an "idle" class**, so the glove stays quiet when the hand is simply moving, instead of always reporting its best guess.
- **Closing the loop** — sending recognised gestures, not raw readings, to the PC, so the same glove can act as both the air mouse and a set of discrete shortcuts.
