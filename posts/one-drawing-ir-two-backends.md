---
title: One drawing IR, two backends — how nb-make keeps preview and PDF identical
date: 2026-09-10
description: The browser preview and the exported PDF in nb-make are not two renderers that agree. They are one list of drawing ops, drawn twice.
tags: [web, backend, pdf, typescript]
---

The usual failure mode of a "design it, then export it" tool is that the two halves drift. The on-screen preview is HTML or canvas, the export is a PDF library, and the two grow their own opinions about line widths, text baselines and where a dotted rule starts. You notice at the worst moment: after printing.

[nb-make](https://github.com/VirajAnand-02/nb-make) is a browser tool for designing notebook pages, ordering them into a notebook, imposing them onto sheets and exporting a print-ready PDF. I built it around one decision that removed that whole class of bug.

## Everything compiles to drawing ops

No component draws anything directly. A page — its ruling, its blocks, a generated calendar, a LaTeX formula — compiles to a flat-ish tree of drawing operations in millimetres, origin top-left, Y increasing downward. That tree is the only thing the renderers understand:

```ts
// src/lib/render/ops.ts
export interface GroupOp {
  kind: 'group';
  ops: Op[];
  matrix?: Matrix;
  opacity?: number;
  /** Axis-aligned clip applied in the group's *own* coordinate space. */
  clip?: { x: number; y: number; w: number; h: number };
}

export type Op =
  | LineOp
  | RectOp
  | EllipseOp
  | PolylineOp
  | PathOp
  | TextOp
  | ImageOp
  | GroupOp;
```

Seven primitives and a group. The group carries an affine matrix and a clip, which is all a nested pattern area or a rotated imposition slot needs.

Two backends consume it. `render/svg.tsx` draws the on-screen preview; `render/pdf.ts` draws the export through pdf-lib. The preview is not an approximation of the output — it is the same ops, drawn by a different backend.

The part that actually makes them agree is text. Text is where renderers diverge, because measuring a string is a font problem, not a geometry problem. Both backends measure with the same AFM metrics for the standard-14 fonts that pdf-lib uses (`render/fonts.ts`), and share the anchoring code (`render/text.ts`). If a label is centred in the preview, it is centred by the same arithmetic in the PDF.

Millimetres everywhere, with the group's matrix handling nesting, kills a second class of bug. There is no pixel-to-point conversion sitting between what you dragged and what got printed.

## Pages are stored proportionally

The second decision: block rectangles are stored as fractions of the page's content box, not absolute positions.

That sounds like a small thing. It means retargeting a notebook from A5 to A6 is just compiling the same designs against a different page size — every block lands in the same relative place, and the rulings regenerate at the new size instead of being scaled bitmaps of the old one.

Only one thing does not follow automatically: type. Halving a page and halving the font size are not the same decision, so the type scale is an explicit per-design setting rather than something inferred. Everything else falls out of the compile.

## The parts this makes easy

Because the renderer is isomorphic — the same code runs in the browser and in Node — the PDF is built in the tab you are looking at. There is no render server, no queue, no upload of your notebook to produce a file. That also holds for a notebook you copied from someone else in the community section: you get the ops, so you get the PDF.

Imposition then becomes pure geometry on top of the IR. A saddle-stitch booklet is a page order plus a transform per slot; cut-and-stack is a different order and the same slot machinery; printer's marks are just more ops. None of it needs to know what a "page design" is.

There are two places the server still earns its keep:

- **LaTeX.** MathJax renders a formula to a vector blob in em units on the server (`lib/latex/mathjax.server.ts`), which the client caches and places like any other block. Line breaking and the page-fit strategies stay on the client, in `lib/latex/layout.ts`.
- **Sync.** Writes are brokered so the browser's unload beacon can authenticate on the way out.

## Local-first was easier than it sounds

The app runs with no configuration at all: notebooks in `localStorage`, images in IndexedDB, PDFs generated in the tab. Supabase is additive. An account gives you sync, publishing to the community, ratings and saves — but local storage stays the primary, synchronous write, so the editor never waits on the network, and an auth outage degrades to "no sync" instead of "no app".

Authorisation lives in Postgres row-level security rather than in application code, which has one trap worth repeating. RLS cannot compare the old and new row, so an update policy that lets you edit your own profile also lets you set `role: admin` on it. That column is guarded by a trigger (`guard_profile_privileges`) that resets it unless the caller is already an admin or the service role.

## What I would keep

The IR is the part I would carry to any other project of this shape. It is one small file of types and matrix helpers, and it converted "does the preview match the print?" from a testing problem into something that cannot really happen. Every feature on top — 16 rulings, parametric generators, mirrored facing pages, imposition — is written against ops, not against a renderer.

nb-make is live at [nb-make.vrj02.dev](https://nb-make.vrj02.dev), and the source is on [GitHub](https://github.com/VirajAnand-02/nb-make).
