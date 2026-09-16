---
title: Picking a face-match threshold with data, not vibes — notes from CBIS
date: 2026-02-18
description: CBIS is a content-based image search system split into six small model services. The most useful afternoon of the project was replacing a guessed similarity threshold with one chosen from 4,753 labelled pairs.
tags: [ai, backend, pgvector, python]
---

CBIS (content-based image search) lets you search a photo library by what is *in* the photos — "receipt from a restaurant", "people at the beach", a specific person — instead of by filename or date. It is a Next.js app in front of six Python model services and a Postgres database with pgvector.

This post covers how the system is split up, and the part I'd most recommend copying: choosing the face-recognition threshold from measured data.

## Six services, one job each

Every model runs as its own FastAPI/uvicorn service with its own environment:

| Service | Port | Does |
| --- | --- | --- |
| CLIP | 8000 | image embeddings + captions, and `POST /encode/text` for queries |
| Type Router V2 | 8001 | classifies what *kind* of image this is |
| NIMA | 8002 | aesthetic quality score |
| Search Pipeline | 8003 | query optimisation + routing (`POST /pipeline`) |
| OCR | 8004 | text extraction, only for images routed as documents |
| Face Detection | 8005 | RetinaFace detection + ArcFace recognition |

The split is not microservices for their own sake. The models have genuinely incompatible dependency stacks — CLIP, NIMA, ArcFace and the OCR model each live in their own conda environment — and very different costs. CLIP and OCR run on the GPU, while NIMA runs on CPU. Separate processes let each one be started, restarted and resourced on its own.

## Route first, then spend compute

The type router is what keeps ingestion from running every model on every image. It is a one-vs-rest random forest trained on CLIP features, so it is multi-label: one image can carry several type labels at once. Expensive work is gated on those labels — OCR, for example, only runs when an image is flagged `is_document`.

The CLIP embedding is computed anyway for search, so routing on top of it costs very little extra.

## Search is a vector query with a detour

A text query takes this path:

```text
User query
  -> CLIP /encode/text          (8000)
  -> Search Pipeline /pipeline  (8003)
  -> pgvector similarity search
  -> ranked results
```

The Next.js side (`lib/search-manager.ts`) owns the orchestration: encode the query with CLIP, hand it to the pipeline service to optimise and pick a route, then run the vector search through Prisma and raw SQL. For documents there is a hybrid path that combines OCR text matches with the vector search.

An honest note on where this stands: the pipeline service's advanced query optimisation was still running in its placeholder mode at the time of the last performance report, and I had not yet run a labelled retrieval benchmark (Precision@10, nDCG and so on). The architecture is in place; the retrieval numbers are the next thing to measure, not something to claim.

## The threshold problem

Face recognition in CBIS works the standard way. RetinaFace finds faces, ArcFace turns each one into an embedding, and a new face joins an existing person if its cosine similarity to that person clears a threshold τ — otherwise it starts a new person.

Everything about the user experience hangs on τ. Too low and two different people merge into one identity; too high and the same person splits into several. I had started with a number that "looked about right", which is to say, no number at all.

So I measured it. Using a small labelled set:

- 5 people, 98 face embeddings
- 1,249 positive pairs (same person)
- 3,504 negative pairs (different people)
- 0 images that failed to embed

The similarity distributions overlap, which is the whole problem in one sentence:

| | mean | std |
| --- | --- | --- |
| same person | 0.533 | 0.168 |
| different people | 0.253 | 0.204 |

With the overlap that wide, there is no threshold that is simply correct — only trade-offs. Sweeping τ gave three defensible operating points:

| Criterion | τ | Result |
| --- | --- | --- |
| Best F1 | **0.365** | F1 0.720 · precision 0.619 · recall 0.858 |
| Best balanced accuracy / Youden's J | 0.330 | balanced acc. 0.839 · J 0.677 |
| Equal error rate (approx.) | 0.390 | FPR 0.176 · FNR 0.181 |

I deployed **τ = 0.365**, the best-F1 point. The reasoning is about which mistake is cheaper *for this app*. A missed match (the same person split in two) is easy for a user to fix by merging. A false match silently mixes someone else's photos into a person's album, which is worse. Going lower towards 0.33 buys recall at the cost of more false merges; going higher towards 0.39 does the opposite.

The deployed config alongside it: a minimum face size of 40 and a minimum detector confidence of 0.6, so tiny or doubtful detections never get the chance to create a bogus identity.

Five people is a small dataset and I would not quote these numbers as general truth about ArcFace. What I would recommend is the *process*: collect labelled pairs, plot both distributions, and pick τ from a criterion you can explain.

## Queues fixed bulk ingestion

The first version called NIMA and face detection synchronously during preprocessing. That was fine for one photo and fell over during bulk imports, where slow model calls turned into request timeouts. Both now go through a queued dispatch with an async worker; the face service exposes `/queue/stats` so the app can see the backlog. In the last live check every service was up and the queue was draining with zero errors.

## What I'd change

- **Run the retrieval benchmark** before any more feature work, so search changes are measured the way the face threshold was.
- **Collect more identities.** Five people is enough to find a sensible τ, not enough to trust the third decimal place.
- **Containerise again.** The services started life in Docker, moved to plain conda processes with a start script during development, and should go back to containers for anything beyond one machine.
