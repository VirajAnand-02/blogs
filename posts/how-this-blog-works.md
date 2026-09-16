---
title: How the blog on this site works
date: 2026-09-17
description: Markdown in its own repo, a build script that turns it into JSON, a few static pages for link previews, and a webhook instead of GitHub Actions. Here's the whole pipeline.
tags: [web, meta, cloudflare, typescript]
---

My portfolio is a single-page React app. It has a terminal you can type into, a command palette, and an arcade that nobody asked for. What it didn't have was a place to write. So I built one, and this post is about how it works, including the parts I changed my mind on halfway through.

The short version: posts are Markdown files in a separate GitHub repo. When I push one, Cloudflare Pages rebuilds the site, a script turns every post into pre-rendered HTML, and the React app reads that at runtime. No CMS, no database, no server.

## Why the posts live in their own repo

The first question was where the posts should live. The obvious answer is a `posts/` folder inside the portfolio repo. I went the other way and put them in [VirajAnand-02/blogs](https://github.com/VirajAnand-02/blogs).

The reason is mostly how it feels to write. When the posts sit next to the site code, every post is also a site commit, and every site change shows up between my posts. Keeping them apart means the blog repo is just writing. Open a file, write, push. I never have to think about React while I'm doing it.

It also means the site doesn't need to know what posts exist. It finds out at build time.

A post is a Markdown file with a bit of frontmatter on top:

```md
---
title: How the blog on this site works
date: 2026-09-17
description: One line for cards and link previews
tags: [web, meta]
draft: false
---
```

The file name becomes the URL, so `posts/how-this-blog-works.md` ends up at `/blog/how-this-blog-works`. Only `title` and `date` are required. If I skip `description`, the first paragraph gets used instead, cut down to 180 characters.

## Getting the posts into the build

Everything starts in `scripts/build-blog.ts`, which runs before Vite. Its first job is finding the posts. On Cloudflare it does a shallow clone of the blog repo:

```ts
execFileSync('git', ['clone', '--depth', '1', '--branch', BLOG_BRANCH, `https://github.com/${BLOG_REPO}.git`, CACHE_DIR], {
  stdio: ['ignore', 'ignore', 'pipe'],
  env: { ...process.env, GIT_TERMINAL_PROMPT: '0' },
});
```

That `GIT_TERMINAL_PROMPT: '0'` line matters more than it looks. Without it, if the repo is missing or private, git politely asks for a username and waits. In a build container, nobody is ever going to answer, so the build just hangs until it times out. With the prompt turned off, it fails straight away with an actual error message.

What happens when the clone fails depends on where the build is running. On Cloudflare, the build stops. I would much rather have a failed deploy than a live site where the blog has quietly gone empty. On my laptop it only prints a warning and carries on with no posts, because I don't want a flaky network to stop me working on the rest of the site.

For writing locally, I skip the clone entirely. Setting `BLOG_DIR=../blogs` points the script at my own checkout, so I can see a post on the dev server before it goes anywhere.

## Markdown to HTML, once

Each post goes through a unified pipeline: remark parses the Markdown (with GitHub-flavoured extras like tables), then rehype turns it into HTML. Along the way, a few small plugins of my own do the fiddly bits.

**Syntax highlighting happens at build time.** Shiki colours the code blocks using the same Catppuccin Mocha theme as the rest of the site. The output is plain HTML with inline colours, so no highlighting library ever ships to the browser. Readers download coloured text, not a tokenizer.

**Headings become a table of contents.** Every `h2` and `h3` gets an id and a `#` anchor, and gets collected into a list. The post page uses that list for the sidebar that tracks where you are.

**Relative links get fixed.** I want to write posts the way I'd write any Markdown in a repo, with images next to the file and links to other posts by file name. The browser has no idea what `./images/diagram.png` or `./other-post.md` means once it's all a website, so a plugin rewrites them. Images get copied into `public/blog/assets/`, and links to `.md` files turn into `/blog/<slug>` links. External links get `target="_blank"` while it's at it.

The copying has a guard, because a path like `../../secrets.txt` is technically "relative" too:

```ts
const rel = path.relative(postsDir, abs);
if (rel.startsWith('..') || path.isAbsolute(rel)) {
  console.warn(`[blog] WARN: asset outside posts/ ignored: ${url}`);
  return url;
}
```

It's my own repo, so this is less about attackers and more about me making a typo and wondering why a random file ended up on the site.

**Mistakes fail loudly.** A post with no title, a date that isn't a date, two files that end up with the same slug, or a slug like `posts` or `rss` that would collide with the build output all stop the build with a message naming the file. Drafts (`draft: true`) are skipped unless I set `BLOG_DRAFTS=1`.

The output is boring on purpose:

- `public/blog/index.json` holds every post's metadata, newest first.
- `public/blog/posts/<slug>.json` holds one post's HTML and its table of contents.
- `public/blog/rss.xml` is a normal RSS feed.

Reading time is worked out here too: the word count with code blocks stripped out, divided by 220. Nobody reads a code block at 220 words a minute.

## The React side is small

Because all the heavy lifting already happened, the client code is mostly fetching JSON. `src/blog/data.ts` is about 50 lines. It fetches the index once, caches each post as it's requested, and that's it.

```ts
let indexPromise: Promise<PostMeta[]> | null = null;
export const getIndex = () => (indexPromise ??= fetchJson<PostMeta[]>('/blog/index.json').then((p) => p ?? []));
```

Caching the promise rather than the result means that if three components ask for the index at the same moment, there's still only one request.

The site has a tiny router I wrote for `/`, `/blog` and `/blog/:slug`. When you click a link inside a post that points at another post, it stays inside the app instead of reloading the page. Code blocks get a copy button added after the HTML is on the page.

Since the rest of the site is built around a terminal, the blog is wired into it as well. `blog` lists posts, `blog ai` filters by tag, and `read 2` or `read <slug>` opens one. Once the index has loaded, the slugs and tags become tab completions. The command palette (Ctrl K) lists every post too.

## The catch with single-page apps: link previews

Everything above worked in the browser. But a blog post mostly travels as a link someone pastes into a chat, and a pasted link to a single-page app gets a pretty sad preview: the homepage title and no description.

Every URL serves the same `index.html`, and the real title only appears after JavaScript runs. LinkedIn, X, Slack and Discord don't run JavaScript when they build a preview. They read the HTML and leave.

The fix is `scripts/prerender-blog.ts`, which runs after `vite build`. For every post, it takes the built `index.html` and makes a copy at `dist/blog/<slug>/index.html` with that post's own tags swapped in:

- a real `<title>` and meta description
- a canonical URL
- Open Graph and Twitter card tags, with the cover image if the post has one
- `article:published_time` and a tag for each post tag

It also puts the whole article inside a `<noscript>` block, so crawlers that don't run JavaScript still see the actual text.

The browser still loads the same React app from that page, so nothing changes for people. Previews just finally show the right thing. One small catch: social sites won't use SVG images for previews, so if a post's cover is an SVG, it falls back to my profile photo.

## Publishing: a webhook instead of GitHub Actions

The last piece is getting a push to the blog repo to rebuild the site. Cloudflare Pages is connected to the portfolio repo, not the blog repo, so it has no idea when I publish a post.

My first version used a GitHub Action. On every push to `posts/`, it sent a POST request to a Cloudflare Pages deploy hook. It was about twenty lines of YAML and it worked fine.

Then a billing problem on my GitHub account meant Actions wouldn't run at all.

The replacement turned out to be simpler. A deploy hook is just a URL that starts a build when something POSTs to it, and GitHub can already send a POST on every push without any Actions involved. That's a plain repository webhook. So now:

1. I push a post to the blog repo.
2. GitHub's webhook POSTs to the Cloudflare deploy hook.
3. Cloudflare runs `npm run build`, which clones the blog repo and does everything above.
4. About a minute later, the post is live.

There's no workflow file to maintain and no secret stored in the repo. The only downside is that the webhook fires on every push, not only when posts change. The blog repo is basically just posts and a README, so that doesn't matter.

## Would I do it the same way again?

Mostly, yes. The best decision was doing the work at build time. The browser gets pre-rendered HTML with the highlighting already done, which keeps the site fast and the client code tiny.

If I started again, I'd plan the prerendering from day one. It's easy to treat link previews as an afterthought, but a blog post mostly travels as a link someone pastes somewhere. If the preview is wrong, most people never click.

And this post is its own test. It's a Markdown file in the blog repo, and if you're reading it on the site, the push, the webhook, the clone, the build and the prerender all worked.
