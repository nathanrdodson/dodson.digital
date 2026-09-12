# Issues and improvements

Running backlog of known problems and planned improvements for dodson.digital. This is the single source of truth for "what's wrong / what's next" — architecture and conventions live in [CLAUDE.md](CLAUDE.md).

This backlog was seeded on 2026-09-12 from [dodson.mba](https://github.com/nathanrdodson/dodson.mba)'s `ISSUES.md`, carrying forward the entries that concern the blog posts, photo albums, and images that moved to this repo in the personal/professional site split. IDs and measurements below are as they stood in the old repo's build — **re-measure against this repo's own build before acting on them**, since the new repo has different content boundaries (no career pages, no `/experience`) and hasn't been built or deployed yet.

## Maintaining this file

- **Log it here when you find it.** Anything you notice but don't fix in the same change gets an entry, so it isn't rediscovered from scratch later.
- **IDs are stable and never reused.** New entries take the next number in their prefix. Reference them in commits and PRs (`Fixes PERF-2`).
- **Record the evidence, not just the claim.** Measured numbers and the command that produced them are the expensive part — keep them so the next person doesn't re-derive them.
- **When you fix something, move the entry to [Resolved](#resolved)** with the date and commit. Don't delete it; the history is why a decision looks the way it does.
- Priorities: **P1** user-visible breakage or major regression risk · **P2** meaningful quality/performance/SEO gain · **P3** hygiene and polish.

---

## Open

### P2

#### `PERF-1` — Images bypass Astro's image pipeline entirely

All images live in `public/`, so `astro:assets` never processes them. They ship as original-resolution JPEGs with no modern formats and no responsive variants; a phone downloads the same 2000px file as a desktop.

- **Measured (in the old repo, pre-split):** 435 JPEG/JPG, 4 PNG, 1 WebP, ~176MB total. Typical 2000×1333 at ~400KB, largest 1.5MB. Per-page payload: `/photos/tunisia` 31.7MB (60 images) · `/photos/charleston-sc` 17.7MB (41) · `/photos/new-mexico` 13.4MB (31) · `/photos/seeds-colorado` 11.2MB (30). All of this content now lives in this repo — re-measure here.
- **Mitigation already in place:** `loading="lazy"` on gallery images.
- **Fix:** move images to `src/assets/` and use `<Image>`/`<Picture>` for automatic AVIF/WebP + `srcset`. Large migration — every content path changes, and the `images:` arrays in `src/content/photos/*.md` must move to `image()` schema refs. Do it on its own branch.
- **Also fixes:** `CI-4` (artifact size) and is the main reason this repo, not dodson.mba, now carries the unbounded-growth risk.

#### `PERF-2` — No `width`/`height` on any image → layout shift

Zero `width=` or `height=` attributes sitewide, so every image reflows the page as it loads.

- **Fix:** cheapest Core Web Vitals win available, and **does not require `PERF-1`** — dimensions can be emitted from the source files at build time while images stay in `public/`.

#### `PERF-3` — Two render-blocking third-party stylesheets

`BaseLayout.astro` loads Google Fonts and Iconoir from jsDelivr in `<head>`. Both block first paint on a third-party connection.

- **Where:** [src/layouts/BaseLayout.astro](src/layouts/BaseLayout.astro)
- **Detail:** Google Fonts has `preconnect`; jsDelivr does not. Iconoir is pinned to `@main`, a moving upstream target — the icon set can change without a commit here.
- **Fix:** self-host fonts via Fontsource and subset Iconoir to the icons actually used. Removes both third-party round trips and the `@main` risk together. Fixing it here won't fix dodson.mba's copy — they're separate repos now.

#### `SEO-1` — Most posts share an identical meta description

Only a couple of posts set `excerpt`, so the rest fall back to the site tagline. This propagates to `og:description` and `twitter:description`, making shared post links look identical.

- **Verify:** `grep -h -oE '<meta name="description" content="[^"]*"' dist/blog/*/index.html | sort | uniq -c`
- **Fix:** in `BaseLayout.astro`, fall back to the first ~155 characters of the post body rather than the site default.

#### `SEO-2` — No sitemap, robots.txt, or RSS feed

None of `sitemap.xml`, `robots.txt`, `rss.xml` exist in the build. A photo/travel blog with no feed is the notable omission.

- **Fix:** add `@astrojs/sitemap` and `@astrojs/rss`; commit a `public/robots.txt` pointing at the sitemap.

#### `CI-1` — No PR validation; a broken build is discovered in production

No CI is configured at all yet in this repo (deploy pipeline is still pending — see CLAUDE.md).

- **Fix:** stand this up alongside the Cloudflare Pages deploy config — a `pull_request` trigger running `npm run build` and `npm run lint:md`, not just a deploy-on-push job.

#### `CI-2` — Nothing enforces linting or type-checking

`tsconfig.json` extends `astro/tsconfigs/strict`, but no type check has ever run — `@astrojs/check` and `typescript` are not installed. `lint:md` exists but no workflow calls it yet.

- **Fix:** install `@astrojs/check` + `typescript`, add `npm run check`, and run it alongside the build in `CI-1`.

#### `A11Y-1` — Album photos cannot have alt text at all

The `photos` schema types `images` as `z.array(z.string())`, so there is nowhere to put alt text for the album photos. They all render `alt=""`.

- **Where:** [src/content.config.ts](src/content.config.ts)
- **Fix:** widen the schema to accept `string | { src: string; alt: string }` and keep plain strings working, then backfill alt text per album.

---

### P3

#### `SEO-3` — `og:image` serves the full-resolution original

Social cards point at the untouched 2000px source (up to 1.5MB). Scrapers want roughly 1200×630.

- **Fix:** generate a dedicated OG derivative. Naturally falls out of `PERF-1`.

#### `SEO-4` — No structured data

No JSON-LD anywhere. `BlogPosting` on posts is the useful one here (no `Person`/professional schema — that belongs on dodson.mba).

#### `CI-3` — No dependency automation

No dependabot or renovate config yet.

#### `CI-4` — Large artifact on every deploy

~176MB (pre-split measurement) of images uploaded/deployed on every build. Resolved as a side effect of `PERF-1`.

#### `DX-2` — No formatter or linter outside Markdown

No prettier, eslint, stylelint, or `.editorconfig`. Markdown is the only linted format in a repo that is mostly Astro, TypeScript, and SCSS.

#### `A11Y-2` — No skip-to-content link

No skip link on any page, so keyboard users traverse the full header and nav on every navigation.

#### `CONTENT-2` — Markdownlint findings in post prose

In the old repo, `npm run lint:md` reported 9 issues across 4 posts (setext headings in `charleston-sc.md`, a duplicate heading in `its-been-a-while-airport.md`, non-descriptive `[here]` link text and blockquote-spacing warnings in `seeds-leadership-2022.md`) — all of which moved here with the content. Re-run `npm run lint:md` to confirm.

- **Deliberately unfixed.** These are personal essays and the lint config exists for structural hygiene, not prose editing. Fix only with the author's sign-off — `lint:md:fix` is not automatically safe here.

#### `CONTENT-3` — Byte-identical duplicate images

In the old repo, ~5MB was wasted on 10 pairs of byte-identical images (same photo referenced under two names) — this content moved here. Re-verify:

- **Verify:** `find public/images -type f -exec md5 -q {} \; | sort | uniq -d`
- **Note:** both copies are referenced from content, so deduping means editing content references. Low value; bundle it into `PERF-1` rather than doing it alone.

#### `DX-3` — Swiper CSS loads on all pages

Slider CSS ships to pages with no slider. The JS is correctly scoped and does not load.

---

## Resolved

| ID | Issue | Resolved | Commit |
| :--- | :--- | :--- | :--- |
| `CONTENT-0` | (inherited from dodson.mba, pre-split) 16 gallery photos 404ing in production after a cleanup pass scanned only `featureImage` and missed the `images:` arrays | 2026-08-15 | `e5bb55c` (dodson.mba) |
| `CONTENT-4` | (inherited from dodson.mba, pre-split) Broken in-page anchor `#map` in the Tunisia post; heading id is `mapbox` | 2026-08-15 | `9db6657` (dodson.mba) |
| `DX-1` | No `.nvmrc`; added one pinning Node 22, needed for the new deploy workflow to pin its Node setup step anyway | 2026-09-12 | `19a88a6` |
