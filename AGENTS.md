# AGENTS.md

Briefing for coding agents working in this repo. Read it before touching
anything — the commands and gotchas here cost real time to relearn.

## Project overview

A personal blog at **wslbn.space**. Posts are Markdown files under
`content/posts/`, rendered at build time by `@nuxt/content` v3 (MDC + Shiki)
into a fully static site, served by Caddy in Docker (exposed via a
Cloudflare Tunnel) or published to Cloudflare Workers. There is no admin UI
and no database — authoring is committing a `.md` file.

## Setup

After a fresh clone:

```bash
# Install deps and bring up the dev server
npm install
npm run dev
# → http://localhost:3000
```

Local preview via Docker (no native Node needed):

```bash
docker compose up --build blog
# → http://localhost:8080
```

Generate the static site (what both deploy targets run):

```bash
npm run generate
# → .output/public/
```

Preview the Workers deployment locally (needs `npm run generate` first):

```bash
npm run wrangler:dev
# → http://localhost:8787
```

## Environment

```bash
# .env.example — copy to .env; Portainer stack env works the same way.

# Cloudflare tunnel token. From Cloudflare Zero Trust → Networks → Tunnels
# → your tunnel → "Install connector". Long base64 blob starting with "eyJ".
# dev: optional, only needed when running -f docker-compose.cloudflared.yml locally
# prod: required, set in Portainer stack env
TUNNEL_TOKEN=
```

Docker run-time env (set via Portainer stack env, not committed):

- `TUNNEL_TOKEN` — see above.

No secrets in the repo. `TUNNEL_TOKEN` is the only runtime env var; the
Workers deploy authenticates through Cloudflare's GitHub App, so it needs
no repo secrets either.

## AI content generation (READ BEFORE WRITING POSTS)

This repo is designed to be writable both by hand and by AI agents. The
rules below keep AI-generated posts indistinguishable in voice and verifiable
in fact. **An AI authoring a post must follow them.**

### Front matter (required, exact form)

Every post file begins with this front matter. Schema is enforced by
`content.config.ts` — wrong fields fail the build, not silently.

```yaml
---
title: "Specific Title Without Clickbait"      # max 60 chars, no emoji
description: "One sentence, 120–160 chars."      # used for SEO + OG + index card
date: "YYYY-MM-DD"                                # ISO, no timezone suffix
tags: ["topic", "subtopic"]                       # lowercase, kebab-case, 1–5
draft: false                                      # true keeps it off the site
---
```

- `title` — sentence-case, no period at the end, no emoji, no ALL CAPS.
- `description` — exactly one full sentence with period. Targets ~140 chars;
  Google truncates ~155.
- `date` — sorted DESC on the index; wrong format breaks ordering.
- `tags` — re-use existing tags from other posts before inventing new ones.
  Mismatched casing creates duplicate buckets.
- `draft: true` — post renders in `npm run dev` (with `-D`) but **not** in
  the production site and **not** in `queryCollection('posts')
  .where('draft', '=', false)`. Use this for WIP; flip to `false` when ready.

### Filename and slug

- Filename: `content/posts/<kebab-case-slug>.md`.
- Slug becomes the URL (`/posts/<slug>/`). Changing the filename after the
  post has been indexed breaks the permalink — never rename without a
  redirect.
- Only `[a-z0-9-]`. No `.`, no uppercase.

### Body

- Use Markdown structure: `#` title only once (the front-matter `title`
  renders separately; don't repeat it as `#` in the body), `##` and below
  for sections.
- First paragraph is the lead — answer what the post is about. Don't open
  with filler ("In this article, we will…").
- Inline HTML works. Use sparingly. The MDC `<sup>`/`<kbd>`/`<mark>` in the
  sample post shows the pattern.
- Vue components in `components/content/*` are callable from any post via
  MDC: `::component-name{prop=value}` — see `@nuxt/content` docs.

### Fact discipline

- **Do not invent facts.** If uncertain, attribute ("the docs say…",
  "according to <link>") or omit.
- **Do not fabricate quotes, code output, benchmarks, or citations.**
- **Code must run.** Re-run snippets before committing them. Pin versions
  referenced.
- **Link to primary sources** (official docs, RFCs, paper) over secondary
  blog posts. Inline citations, not footnote-heavy style.
- **Match the tone of the existing corpus.** The sample post and any
  previously-merged posts are the reference for voice. If the new post
  reads like a different blog, redo it.

### Internal linking

- Other posts: `[text](/posts/<slug>/)`.
- Site pages: `[About](/about)`.
- Reuse existing tag pages when one applies: `/tags/<tag>` (only resolve
  this if a tags-index page exists — see `pages/tags/...` if added; don't
  assume).

### Before commit — content checklist

- [ ] Front matter validates against `content.config.ts`.
- [ ] `npm run generate` succeeds locally.
- [ ] Post renders at `http://localhost:3000/posts/<slug>/`.
- [ ] No `uncommitted idea`-style TODOs left in body.
- [ ] Description length between 120 and 160 chars.
- [ ] All code blocks were actually executed.
- [ ] All outbound links resolve to live URLs.

### What AI must NOT do

- Don't add front-matter keys outside the schema — the build will fail.
- Don't commit secrets, API keys, tokens, real names/addresses, or anything
  private.
- Don't add a robot.txt that blocks crawling.
- Don't reorganize `content/config/posts` — repos like this are
  CI-tested; unknown filenames or unsupported directories fail the build.

## Development workflow

- Live preview: `npm run dev`. Hot reload picks up `content/posts/*.md`.
- Add a post: create `content/posts/<slug>.md` with the front matter above
  and `draft: false` when ready.
- Add images: drop them under `public/img/` and reference as
  `![alt](/img/<file>)`. SVG preferred for diagrams; raster only when
  needed.
- Add a site page: create `pages/<route>.vue`. Components go under
  `components/<Name>.vue` (auto-imported).
- Tailwind v4 is wired via `@tailwindcss/vite` and `@plugin
  "@tailwindcss/typography"` in `assets/css/main.css`. Add Tailwind
  classes inline; no `tailwind.config.ts` needed for the typography plugin.
- Dark mode is class-based: `@custom-variant dark` in `assets/css/main.css`
  plus `@nuxtjs/color-mode`, which puts `.dark` on `<html>` (dark by
  default, toggled from the header, persisted in localStorage). Pair every
  light class with a `dark:` variant — the dark palette is `stone`, the
  light one stays `neutral`.
- Do not hand-edit `themes/` — this repo does not use a theme; it builds
  the layout in `app.vue` and `components/`.

## Testing

The single real test: build the production bundle and confirm the static
output renders.

```bash
npm run generate
# Inspect output
ls -la .output/public
ls .output/public/posts/hello-world/
# Must contain index.html
```

Smoke-test inside Docker (the compose file publishes no host port —
cloudflared reaches the container over the Docker network, so map one for
the test):

```bash
docker build -t wslbn-space-blog:latest .
docker run -d --rm --name blog-smoke -p 8080:80 wslbn-space-blog:latest
curl -sf http://localhost:8080/ | grep -q "Stuff that is interesting to me"
curl -sf -o /dev/null -w 'HTTP %{http_code} /posts/hello-world/\n' http://localhost:8080/posts/hello-world/
docker rm -f blog-smoke
```

CI runs `docker build` and a smoke test on every push and PR. Green means
the SSG output serves both `/` and `/posts/hello-world/`.

There are no unit tests. Don't add them unless requested.

## Code style

- TypeScript everywhere except `.vue` `<script>` blocks (which use
  `lang="ts"` for any logic).
- 2-space indent, single quotes for strings, no trailing semicolons in
  `.ts`/`.vue` `<script>` blocks.
- Vue components: `<script setup lang="ts">` only. No Options API.
- Naming: components `PascalCase.vue`, pages `kebab-case.vue` or
  `[param].vue`, composables `useThing.ts`.
- Don't introduce a formatter (Prettier) or linter (ESLint) without a real
  request — one-author repo, tooling creep is overhead. Match the existing
  style.

## Build and deployment

Build (production static):

```bash
docker build -t wslbn-space-blog:latest .
```

The image is multi-stage:

- `builder` — `node:24-alpine` + Python/g++ for `better-sqlite3`; runs
  `npm ci` and `npm run generate`.
- final — `caddy:2-alpine` serving `.output/public/` from `/srv` with
  `docker/Caddyfile` (curl added only for the healthcheck).

Deploy (manual via Portainer):

```bash
docker compose -f docker-compose.yml -f docker-compose.cloudflared.yml up -d
```

Stops/restarts the same way. Logs: `docker compose logs -f`.

CI does not deploy the Docker/Tunnel stack — `weselben` redeploys from
Portainer after merging. The CI workflow (`.github/workflows/build.yml`)
only builds + smoke-tests; it exists so PRs can't land broken content.
The Workers publish runs on Cloudflare's side (see below).

### Cloudflare Workers (auto-deploy on main)

The site publishes to Cloudflare Workers through **Workers Builds**: the
Cloudflare dashboard holds the GitHub App connection and runs
`npm run generate` + `wrangler deploy` on every push to `main`. Deploy
config lives in `wrangler.jsonc` — assets-only, no worker script, same
pretty-URL handling as Caddy. No repo secrets and no Actions workflow
involved; the connection is managed in the Cloudflare dashboard
(Workers & Pages → the worker → Settings → Build).

Manual deploy from a machine works too: `npx wrangler login` once, then
`npm run generate && npm run wrangler:deploy`.

The Docker/Tunnel path stays the primary setup; Workers is a second
publish target built from the same `npm run generate` output.

Public hostname routing: in Cloudflare Zero Trust → Tunnels → your tunnel
→ Public Hostnames, entry must be:

- hostname: `wslbn.space`
- service: `http://blog:80`

Not `http://localhost:80` — that resolves to the `cloudflared` container.

## Pull requests

- Title: Conventional Commits — `<type>(<scope>): <summary>` (`feat: …`,
  `fix: …`, `chore: …`, `docs: …`). Applies to PR titles and
  to the squash commit subject alike.
- **Posts use `post:`.** Any commit touching `content/posts/*` — new posts
  *and* edits to existing ones — commits as `post: <summary>`, never
  `feat:`, `docs:` or `content:`. Example: `post: packaging gomodel for
  arch linux`. Site code/docs keep the standard types above.
- **Squash-merge only.** One commit lands on `main` per PR — never a merge
  commit, never a rebase pile. Enforced in the repo's merge settings (only
  "Squash and merge" is enabled).
- **Never commit directly to `main`.** Branch protection rejects direct
  pushes — every change lands via PR, no exceptions (including the owner).
  Required status check: `docker-build`. Enforced for admins too. Branch
  off `main`, push, open **draft** PR with `gh pr create --draft`.
- Don't merge own PRs.
- One concern per PR. Content-only PRs (just a new post) are welcome and
  can skip the full template — title + 1-sentence body is enough.

## Security and secrets

- Never commit `TUNNEL_TOKEN`. It lives in `.env` (gitignored) or Portainer
  stack env, never in the file tree.
- Rotate the tunnel token if exposed (Cloudflare dashboard → Tunnels →
  Reset token).
- Sanitize pasted content: do not paste API keys, real names, addresses,
  or anything private into posts. Even "redacting" is error-prone — strip
  at source.

## Gotchas

- **Nuxt 4 file layout.** `pages/`, `components/`, `app.vue`,
  `content.config.ts`, `nuxt.config.ts`, `assets/css/main.css` all sit at
  the repo root. Don't add `src/` wrapper.
- **`.data/`** — `@nuxt/content` v3 stores a SQLite cache under `.data/`.
  Gitignored. Don't commit; CI regenerates.
- **Schema is enforced.** Adding a new field to a post requires both
  adding it to `content.config.ts` and documenting the rule here first.
  Otherwise the build fails with a Zod error.
- **`draft: true` ≠ hide in dev.** `npm run dev` (with `-D` shorthand
  flag) shows drafts. `npm run generate` does **not**. Confirmed in
  `pages/index.vue` — production uses `.where('draft', '=', false)`.
- **CNAME collision.** `public/CNAME` must contain exactly `wslbn.space`.
  Don't add trailing newline gymnastics — one literal line.
- **`better-sqlite3` on Alpine.** Already handled — Dockerfile installs
  `python3 make g++` before `npm ci`. Don't remove that line without
  retesting in CI.
- **npm install scripts are allowlisted.** `package.json` has an
  `allowScripts` block (`better-sqlite3`, `esbuild`). After adding a
  dependency with an install script, run `npm install-scripts approve <pkg>`
  and commit the `allowScripts` change — builds fail with missing bindings
  otherwise, in CI and Docker alike.
- **Lockfile must stay npm-10.9.2-authored.** Cloudflare's Workers Builds
  image pins `npm@10.9.2` and runs its own `npm clean-install` before the
  build command; the Docker builder pins the same version (node:24's
  bundled npm 11 rejects the lockfile). Regenerate the lockfile with
  npm 10.9.2 — e.g.
  `npx -y npm@10.9.2 install --package-lock-only` inside `node:24-alpine`
  — never with the host npm. `oxc-parser` is pinned as a devDependency on
  purpose: as an optional peer dep it otherwise disappears from the
  lockfile and every npm version's `npm ci` then complains about missing
  entries.
- **Nitro preset is pinned to `static` on purpose.** CF Workers Builds
  injects `WORKERS_CI`; without the explicit preset Nitro flips to
  `cloudflare-module`, emits a `.wrangler/deploy/config.json` redirect and
  a server entry that a static site never produces — the deploy then fails
  on a missing `index.mjs`. `nuxt.config.ts` also deletes a stale
  `.wrangler/deploy/` before every build because the CF build-output cache
  can restore one from an older non-static build. Don't remove either.
- **No email exposure in content.** When posting screenshots or logs that
  contain personal info, scrub them. Even on a private repo, the build
  artifact is public.
- **Tags can fragment.** Coining a new tag on a fresh post (e.g. "infra"
  vs "infrastructure") creates a new index. Reuse what exists; check
  `git grep -i tag: content/posts/` for the in-use tag list before adding.
