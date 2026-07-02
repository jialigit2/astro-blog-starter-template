# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Astro 5 blog starter deployed to **Cloudflare Workers** as a static site. Features: Markdown/MDX, sitemap, RSS feed, SEO/OG metadata. Requires Node `>=22`.

## Commands

All commands run from the repo root.

| Command | Purpose |
| --- | --- |
| `npm install` | Install dependencies |
| `npm run dev` | Local dev server at `localhost:4321` |
| `npm run build` | Production build to `./dist/` (produces `_worker.js` + static assets) |
| `npm run preview` | Build then run via `wrangler dev` to emulate Cloudflare locally |
| `npm run check` | Full pre-deploy validation: `astro build && tsc && wrangler deploy --dry-run` |
| `npm run deploy` | Deploy to Cloudflare Workers via `wrangler deploy` |
| `npm run cf-typegen` | Regenerate `worker-configuration.d.ts` from the latest `Env` bindings |
| `npx astro check` | Type-check `.astro` files |
| `npx astro add <integration>` | Add a new integration (updates `astro.config.mjs` + deps) |

There is no test suite configured in this template.

## Architecture

### Build & deploy pipeline
- [`astro.config.mjs`](astro.config.mjs) wires three integrations: `@astrojs/mdx`, `@astrojs/sitemap`, and the `@astrojs/cloudflare` adapter (`platformProxy.enabled = true`).
- [`wrangler.json`](wrangler.json) points the Worker entry at `./dist/_worker.js/index.js` and serves static files from `./dist` via the `ASSETS` binding. `nodejs_compat` is enabled; observability is on; source maps upload.
- The deploy order matters: `astro build` must produce `dist/` before `wrangler deploy` (or `wrangler dev`) can find the entry.

### Content collection
- [`src/content.config.ts`](src/content.config.ts) defines a single `blog` collection using the `glob` loader over `./src/content/blog/**/*.{md,mdx}` with a Zod schema (`title`, `description`, `pubDate`, `updatedDate?`, `heroImage?`).
- Adding/changing frontmatter fields requires editing the schema in this file. `pubDate` and `updatedDate` are coerced to `Date`.
- Posts are accessed via `getCollection('blog')` from `astro:content`; the `id` field is the slug (used in URLs like `/blog/${post.id}/`).

### Routing
- [`src/pages/index.astro`](src/pages/index.astro) — landing page.
- [`src/pages/about.astro`](src/pages/about.astro) — about page.
- [`src/pages/blog/index.astro`](src/pages/blog/index.astro) — post listing (sorted desc by `pubDate`).
- [`src/pages/blog/[...slug].astro`](src/pages/blog/[...slug].astro) — dynamic post route; `getStaticPaths` iterates `getCollection('blog')` and renders via the `BlogPost` layout.
- [`src/pages/rss.xml.js`](src/pages/rss.xml.js) — RSS endpoint exporting `GET(context)` using `@astrojs/rss`.

### Layouts & components
- [`src/layouts/BlogPost.astro`](src/layouts/BlogPost.astro) is the single post template; props are typed as `CollectionEntry<'blog'>['data']` and the post body is injected via the default `<slot />`.
- Shared chrome: `BaseHead.astro` (meta + OG), `Header.astro` + `HeaderLink.astro`, `Footer.astro`, `FormattedDate.astro`. Styles live in [`src/styles/global.css`](src/styles/global.css); component-scoped `<style>` blocks are used in pages and the layout.
- Global site constants (`SITE_TITLE`, `SITE_DESCRIPTION`) live in [`src/consts.ts`](src/consts.ts) and are imported by pages, RSS, and the listing.

### Cloudflare types
- [`src/env.d.ts`](src/env.d.ts) extends `App.Locals` with the `@astrojs/cloudflare` `Runtime<Env>` shape.
- `worker-configuration.d.ts` is generated (do not hand-edit); rerun `npm run cf-typegen` after changing `wrangler.json` bindings.

## Conventions

- Site URL is hard-coded in `astro.config.mjs` as `site: "https://example.com"` — change this for canonical URLs, sitemap, and RSS to resolve correctly.
- Post slugs are derived from the file path under `src/content/blog/`; the file name (minus extension) becomes `post.id`.
- `heroImage` is referenced as a plain string — image files used here should be placed in `public/` so paths resolve at build time.