---
title: A self-correcting Manim compiler, and the 3.4 seconds that shaped it
date: 2026-08-28
description: Manimate turns a topic into a narrated animated video. Most of the engineering went into surviving LLM-generated Python and into not paying Manim's import cost over and over.
tags: [ai, backend, automation, python]
---

[Manimate](https://github.com/VirajAnand-02/manimate_uni) takes a topic, researches it, plans a lecture, writes [Manim](https://www.manim.community/) scene code, renders it, synthesises a voiceover locally and stitches the result into a narrated video. The whole pipeline is orchestrated from a Next.js app.

Two problems dominated the build, and neither was the LLM prompt.

## Problem 1: generated code does not compile

Manim scenes are Python. An LLM writing Python against an API it half-remembers produces code that fails — a wrong constructor argument, a removed method, two scenes colliding on a helper name.

So a failed render is not an error, it is an input. The pipeline captures the traceback, hands it to a corrector model together with the offending script, and re-renders. Up to three attempts, then the scene is marked failed and the job carries on.

This works well because a Python traceback is unusually good feedback: it names the file, the line and the exception. The corrector is not guessing what went wrong.

The security consequence is the obvious one, and worth being blunt about: the pipeline executes LLM-generated Python. The container is the sandbox. Child processes get a minimal environment so generated code never sees the API keys, and the image runs as an unprivileged user. Web-search snippets flow into the code generator, which is a prompt-injection path to code execution — acceptable for an authenticated deployment you control, not acceptable for public sign-ups without real per-render isolation.

## Problem 2: `import manim` costs ~3.4 seconds

That is the line that reshaped the scheduler.

The naive structure — one Manim process per scene — pays that import per scene, and then pays it again for every correction attempt. A ten-scene lecture with a couple of retries spends more time importing than animating.

The fix is to batch: every scene in a module renders in a **single Manim process**, with class names rewritten so they stay unique. Modules then render in parallel up to `MANIM_RENDER_CONCURRENCY` (cores − 1, capped at 4).

Batching creates its own failure mode, though. One syntax error takes down the whole file, and two scenes can still collide on a helper name. So the batch is the fast path, not the only path: scenes the batch did not produce fall back to individual renders with the usual correction loop. Fast when things work, no worse than before when they don't.

Two smaller wins came from the same "stop repeating yourself" instinct:

- **Voiceover does not wait for rendering.** Its input is the lecture plan's text, so Kokoro-82M (ONNX, running locally — no paid TTS API) starts as soon as the plan exists and runs alongside code generation and rendering. Only the final ffmpeg mux needs both halves.
- **Manim's LaTeX and text caches are shared across the whole job**, instead of being thrown away between correction attempts.

## Making narration and animation line up

A scene that is 4 seconds long with 9 seconds of narration is a frozen video track. The other way round is silence.

Scene durations are computed from the voiceover text before rendering:

```
duration = ceil(chars / 15) + 3 seconds
```

Crude, but it is a *prediction*, which is what matters — the animation is generated to fit the narration rather than being cut to it afterwards. ffmpeg then muxes and stitches per module.

## Where state lives

Postgres holds one row per generation in `public.jobs`, with the lecture plan and quiz as `jsonb`. Finished `video.mp4` and the generated `.py` go to a private Supabase Storage bucket under `{user_id}/{job_id}/`. Row-level security scopes every job to its owner, so route handlers never compare user ids by hand. Playback redirects to a short-lived signed URL, so video traffic bypasses the app server while still requiring an authorised request to get the link.

Everything else — `media/`, `tts/`, per-module MP4s — is scratch on local disk under `MANIMATE_WORK_DIR`, deleted when the job ends.

Progress updates go through an `update_job_stage()` function in Postgres (migration `0002`), which merges the stage server-side. One round trip per tick instead of two, and concurrent stage updates stop clobbering each other. The app falls back to a slower client-side merge if that migration is missing, and logs a warning saying so.

## Why it cannot run on Vercel

I wanted it to. It can't: renders take minutes, spawn Python subprocesses and need a writable disk. `POST /api/generate` returns `202` and keeps working in the background, so any platform that suspends the instance after the response kills every job mid-render.

It ships as a Docker image instead — Node, Python + Manim, ffmpeg, a TeX Live subset and the pre-downloaded Kokoro weights, which lands at 3–4 GB. The host requirements are specific: scale-to-zero disabled, at least 2 vCPU and 4 GB RAM (Manim at 720p30 will OOM a 512 MB instance), and exactly **one** instance, because the render queue and the cancellation registry are in-process. Jobs orphaned by a restart are failed at boot by a reaper.

That single-instance constraint is the honest cost of running the queue in memory. It was the right trade for a project where the bottleneck is CPU-bound rendering on one box — but it is the first thing I would tear out to scale it.

The provider layer is deliberately boring in comparison: the Vercel AI SDK in front of OpenAI, Anthropic, Gemini, Mistral, Groq, OpenRouter and NVIDIA NIM, with comma-separated keys rotated round-robin per request for throughput.
