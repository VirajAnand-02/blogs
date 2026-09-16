---
title: Knowing when to stop — dynamic cutoffs in CBIS, a self-hosted image search
date: 2026-09-03
description: Vector search always returns something. CBIS decides how much to return by looking for the elbow in the distance curve, and most of the rest of the system exists to give that search better vectors.
tags: [ai, backend, postgres, python]
---

A vector index never says "no results". Ask pgvector for the 20 nearest images to *"whiteboard with a system diagram"* and you get 20 images, whether your library holds three whiteboards or none. The top few are right, the rest are whatever happened to be least far away, and the UI shows them with the same confidence.

[CBIS](https://github.com/VirajAnand-02/CBIS) (Content-Based Image Search) is a self-hosted search engine for a personal photo library: type what you remember, get the photo back. It uses CLIP for meaning, OCR for text, face recognition for people and NIMA for "the good ones". Of all the parts, the one I would reuse elsewhere is the small SQL function that decides where the results stop.

## The shape of the system

A Next.js app handles uploads, the UI and all the database access through Prisma. The models run as separate FastAPI services, each loaded once and kept warm:

| Service         | Port | Does                                                                |
| --------------- | ---- | ------------------------------------------------------------------- |
| `clip`          | 8000 | CLIP ViT-B/32 image and text embeddings (512-d), BLIP captions      |
| `type-router-v2`| 8001 | Multi-label image classification from the CLIP embedding            |
| `nima`          | 8002 | Aesthetic quality score                                             |
| `search-pipeline`| 8003| Query intent and search strategy                                    |
| `ocr`           | 8004 | docTR text extraction                                               |
| `face-detection`| 8005 | RetinaFace detection + ArcFace embeddings, person matching           |

Everything lands in Postgres with pgvector: one embedding per image, OCR text with its own vector, face instances linked to people, and a row of attributes (`isDocument`, `hasPeople`, `nimaScore`…).

## Ingest: embed once, route by type

The naive pipeline sends every image through every model. That is wasteful. OCR on a sunset takes seconds and finds nothing, and face detection on a screenshot of a terminal does the same.

So an upload goes to CLIP first, and the **type router** decides what else is worth running. The interesting part is its input: it does not look at the image at all. It takes the 512-d CLIP embedding the pipeline already computed, and runs a one-vs-rest random forest over it:

```ts
// apps/next-js/lib/preprocessing-manager.ts
body: JSON.stringify({ embedding, threshold: 0.5 }),
```

Ten labels come out: `is_document`, `is_handwritten`, `has_scene_text`, `has_people_faces`, `is_screenshot`, `is_art_illustration`, `has_machine_code`, `is_natural_image`, `is_nsfw`, `is_low_quality`. A random forest on CLIP features is cheap to train and nearly free to run, because the expensive step is already paid for.

Nobody hand-labelled the training data, either. `dataset_gem.py` sent the training images to Gemini with a fixed multi-label schema and wrote the answers to a CSV. A large multimodal model labels the data once, offline, and a small classifier does the work at runtime. It is a good trade for any "is this a kind of X?" question that runs on every upload.

The predictions then fan out:

- `is_document` → OCR
- `has_people_faces` → face detection
- every image → NIMA

Face detection and NIMA are slow, so they don't hold up the upload. They go onto local dispatch queues with retries and a growing back-off (`FACE_DISPATCH_MAX_ATTEMPTS`, `NIMA_DISPATCH_RETRY_DELAY_MS`). Pending and running jobs are stored in Postgres and restored on boot, so a restart doesn't silently drop half an import.

## Search: three queries, one ranking

A query is encoded with CLIP's *text* encoder, which puts it in the same space as the image embeddings. The `search-pipeline` service reads intent from it ("document", "people", "aesthetic"…) and returns a strategy: which tables to search, attribute filters, a minimum NIMA score, and weights. If that service is down, the search manager uses a default strategy and carries on. Only CLIP is a hard dependency.

Then three searches run in parallel:

1. **Image vectors.** The query vector against CLIP image embeddings.
2. **OCR hybrid.** The query vector against OCR-text vectors, plus a case-insensitive substring match on the OCR text.
3. **OCR text.** A plain `ILIKE` search, skipped for queries shorter than four characters. Those match almost everything.

The results are merged by blob id with weighted scores, then sorted by *source tier* first and score second. A CLIP match beats an OCR-vector match, which beats a bare text match. Adding up scores that came from different models doesn't give you one comparable number. Tiers keep the ordering honest, and the weights only rank results within a tier.

There was one trap in step 2. OCR text is embedded with the CLIP text encoder so that a meaning-based query can find a document, but CLIP's text encoder stops at 77 tokens, and a scanned page produces thousands. Long input fails with a `max_position_embeddings` error. The encoder call trims the text to 40 words and 180 characters, and retries shorter if it still fails. That vector only captures the start of the page, which is roughly what a person remembers about a document anyway. The full text is still covered by the `ILIKE` search.

## The elbow

Back to the opening problem. Both vector searches go through Postgres functions, `cbis_vector_search_elbow` and its OCR twin `cbis_ocr_vector_search_elbow`, instead of a plain `ORDER BY … LIMIT`.

It takes the nearest candidates after filtering, sorted by cosine distance. That list of distances is a non-decreasing curve. When a query has real matches, the curve is flat for a while (similar images at similar distances), then jumps, then flattens again into unrelated images. The function looks for that jump:

```sql
diffs AS (
  SELECT rank, distance,
         distance - LAG(distance) OVER (ORDER BY rank) AS diff
  FROM candidates
),
diff_stats AS (
  SELECT MAX(diff) AS max_diff,
         PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY diff)
           FILTER (WHERE diff IS NOT NULL) AS median_diff
  FROM diffs
),
```

The biggest step counts as an elbow only if it is at least `sharpness` times the median step (3 by default). If so, everything before it is kept. If not, the curve is smooth, there is no natural cluster, and the function falls back to `max_results`. Either way, the count is clamped between `min_results` and `max_results`:

```
keep_n = min(n, max(min_results, min(max_results, elbow_n)))
```

Some things I like about this approach:

- **It adapts per query.** A specific query with a few strong hits returns a short list (never fewer than `min_results`, which is 5). A vague query returns a page. No global similarity threshold has to hold for both.
- **It compares distances to each other, not to a fixed number.** Absolute cosine similarity values drift between models and between query styles, but "this gap is much bigger than the usual gap" does not.
- **The median makes it robust.** One big jump is judged against the typical step, so a noisy tail of far-away candidates can't inflate what counts as "typical".
- **It runs in one SQL statement.** No extra round trip, and filters apply *before* ranking, so the elbow is found in the filtered set you actually asked for.

The cost is that pagination works on the kept set rather than the whole table, so `total_count` means "results worth showing". For search, I think that's the right definition.

## Faces

Face search uses a different kind of cutoff. A face either belongs to a known person or it doesn't, so it needs a fixed similarity threshold rather than an elbow. How that threshold was chosen from labelled pairs, instead of guessed, is its own post: [picking a face-match threshold with data](./cbis-face-threshold-and-services.md).

## What I would change

The dispatch queues and job restore work, but they are a small job system built inside a web server, and the next step is a real queue. The query optimizer is still mostly rule-based. And the elbow's `sharpness` is one global value that should really be tuned per search type, because OCR distances have a different shape from image distances.

The elbow function is the part I'd keep as-is. It's a single SQL function with no extra services, and it made the search results feel much more trustworthy.
