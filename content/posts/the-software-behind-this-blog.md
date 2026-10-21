---
title: "The software behind this blog"
description: "What runs wslbn.space: Nuxt 4 and @nuxt/content turn plain Markdown into a static site, shipped to Cloudflare Workers and a Docker stack behind a tunnel."
date: "2026-10-21"
tags: ["meta"]
draft: false
---

`wslbn.space` is a static site with no admin UI and no database. Every post
is a Markdown file in a Git repository, and two independent deploy paths
ship it to the edge. This post explains the pieces — [hello, wslbn.space](/posts/hello-world/)
covered the writing loop, this one covers the machinery.

## The stack

- **Nuxt 4** with `@nuxt/content` v3 — Markdown files under `content/posts/`
  are parsed at build time into a typed collection. The schema in
  `content.config.ts` enforces the front matter; wrong fields fail the build
  instead of silently breaking the index.
- **Tailwind CSS v4** in CSS-first mode — no config file, just
  `@import "tailwindcss"` plus the typography plugin. Dark mode is a single
  `@custom-variant` and `@nuxtjs/color-mode`, dark by default.
- **Shiki** for syntax highlighting, done at build time — zero highlighting
  JavaScript reaches the browser.

## From Markdown to static output

`npm run generate` prerenders every route into `.output/public/`. Each post
becomes a real `.html` file, so a direct visit paints instantly — no server
rendering at request time, no database queries on the hot path.

Every post starts like this, and the schema above rejects anything else:

```yaml
---
title: "The software behind this blog"
description: "One sentence, 120 to 160 characters, reused for SEO."
date: "2026-10-21"
tags: ["meta"]
draft: false
---
```

## Two deploy paths, one build output

Both targets run the same `npm run generate` and serve the same static
files:

1. **Docker + Cloudflare Tunnel.** A multi-stage image builds the site with
   Node, then hands `.output/public/` to a Caddy container. A Cloudflare
   Tunnel exposes it — no open ports, no public origin server.
2. **Cloudflare Workers Builds.** Merging to `main` triggers a build on
   Cloudflare's infrastructure, which publishes the output as an assets-only
   Worker. Push-to-deploy, zero secrets in the repo.

`main` is protected: changes land through pull requests only, the CI has to
pass a real production build, and merges are squash-only. The homepage
search runs on a prerendered search index — sections of every post ship in
the page payload, and the browser filters them locally with no extra
requests.

## The one sharp edge

The Cloudflare build image pins `npm@10.9.2`, so the lockfile must stay in
the format that version writes — regenerating it with a newer npm breaks
the Cloudflare install. It is documented in `AGENTS.md` together with the
fix, because it cost a day of debugging to find.
