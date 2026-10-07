# blog

Content for [the portfolio](https://github.com/VirajAnand-02): Markdown posts and the Frame 30
magazine issues. Push to `main` and the site rebuilds on Cloudflare Pages.

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

## Publishing a Frame 30 issue

One folder per issue, named `YYYY-MM` — that name becomes the URL `/frame30/<YYYY-MM>`:

```
frame30/
└── 2026-06/
    ├── issue.md      metadata + the editor's note
    ├── issue.pdf     the magazine (always this exact filename)
    ├── cover.webp    page 1, shown on the site
    └── cover.jpg     page 1, used for link previews
```

```md
---
title: सफ़र · Safar
date: 2026-06-01          # the 1st of the issue's month
description: One line for the archive card and link previews.
tags: [photography, safar]
gear: [Fujifilm X-T30, 35mm f/2]   # optional
photos: 24                          # optional
pdfBytes: 37300135                  # written for you by `npm run frame30:cover`
pages: 29                           # written for you by `npm run frame30:cover`
---

The editor's note, in Markdown.
```

Don't write `cover.webp`, `cover.jpg`, `pdfBytes` or `pages` by hand — run `npm run frame30:cover`
from the portfolio repo and it renders the covers from page 1 and fills those fields in.

The PDFs are ~37 MB, which is over Cloudflare Pages' 25 MB asset cap, so the site never serves them:
the reader streams them from `raw.githubusercontent.com` with range requests. That also means the
site build clones this repo *without* the PDFs, so adding issues doesn't slow deploys.

## One-time setup

1. In the Cloudflare Pages project: **Settings → Builds → Deploy hooks → Add deploy hook** (branch: the production branch).
2. In this repo: **Settings → Webhooks → Add webhook**. Payload URL: the hook URL. Content type: `application/json`. Events: **Just the push event**. (GitHub pings the hook once on save, which starts a build.)

The webhook calls the hook on every push, so no GitHub Actions minutes are needed. To redeploy by hand: `curl -X POST <hook URL>`.
