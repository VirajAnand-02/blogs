# blog

Markdown posts for [the portfolio](https://github.com/VirajAnand-02). Push a post to `main` and the site rebuilds on Cloudflare Pages.

## Writing a post

Create `posts/<slug>.md` (the file name becomes the URL `/blog/<slug>`):

```md
---
title: My post title
date: 2026-09-20
description: Optional one-liner for cards and link previews
tags: [backend, ai]
cover: ./images/cover.png   # optional link-preview image — PNG/JPG, ~1200×630 (SVG isn't supported by LinkedIn/X)
draft: false                # true = not published
---

Your post in Markdown (GitHub-flavoured: tables, task lists, code fences…).
```

Images go in `posts/images/` and are referenced with relative paths. Link to another post with its file, e.g. `[see also](./other-post.md)`.

## One-time setup

1. In the Cloudflare Pages project: **Settings → Builds → Deploy hooks → Add deploy hook** (branch: the production branch).
2. In this repo: **Settings → Webhooks → Add webhook**. Payload URL: the hook URL. Content type: `application/json`. Events: **Just the push event**. (GitHub pings the hook once on save, which starts a build.)

The webhook calls the hook on every push, so no GitHub Actions minutes are needed. To redeploy by hand: `curl -X POST <hook URL>`.
