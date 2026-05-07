# JP Dogs CMS — Plan

## Context

You run [jpdogs.com](https://jpdogs.com) — a community photography project documenting dogs in Jamaica Plain. The site is a Hugo static site **hosted on GitHub Pages** (verified: DNS → `185.199.108-111.153`, served by GitHub via Fastly), built and deployed by [.github/workflows/hugo.yaml](.github/workflows/hugo.yaml) on every push to `main`. You shoot with a Nikon Z8, edit in Lightroom Classic, and currently export 1600×1067 JPGs at ~400 KB each. You've published 743 albums (one per dog), totaling ~2.5 GB of source content and a 2.7 GB built site.

Day-to-day, you described it as "super simple but a pain in the butt to manage." The pain is **manual album creation and photo addition**: making the folder, writing `index.md`, renaming files to `{Name}-{YYYYMMDD}-{n}.jpg`, picking a featured image, committing. With 743 albums and growing, this adds up.

This plan describes a **local web-based CMS** that automates that workflow, plus a one-shot image conversion that shrinks the repo by ~85%.

---

## What you have today (verified)

**Stack:**
- Hugo Extended ≥ 0.123, theme `hugo-theme-gallery/v4` vendored at [_vendor/github.com/nicokaiser/hugo-theme-gallery/v4/](_vendor/github.com/nicokaiser/hugo-theme-gallery/v4/)
- 26 layout overrides in [layouts/](layouts/) (custom `home.html`, `featured.html`, `gallery.html`, story templates)
- GitHub Pages deploy via `git push` → GitHub Actions → live in ~1–2 minutes
- Custom domain via [CNAME](CNAME) → [jpdogs.com](https://jpdogs.com)

**Content shape:**

| Type | Location | Count |
|---|---|---|
| Dog albums (Leaf Bundles) | [content/albums/](content/albums/) | 741 |
| Featured single gallery | [content/featured/](content/featured/) | 31 images |
| Open Studios 2024 gallery | [content/openstudios/](content/openstudios/) | 254 images |
| Story pages | [content/story/](content/story/) | a few |
| Static pages | [content/about.html](content/about.html), [content/support.html](content/support.html) | 2 |

**Per-album shape** (representative — [content/albums/acre/](content/albums/acre/)):
```
content/albums/acre/
├── index.md         # YAML frontmatter, ~7 lines
├── Acre-20241224-1.jpg
├── Acre-20241224-2.jpg
└── ...
```
```yaml
date: 2024-05-05
featured_image:        # almost always blank
title: Acre
description:           # blank in 740/741
tags: ["acre"]
```

**Repo size:** 2.5 GB working tree, 3.8 GB `.git` (history holds copies of every photo ever re-committed). Built site is 2.7 GB — over GitHub Pages' 1 GB soft limit, currently unenforced.

---

## How Hugo + your theme actually behave (the constraints)

Verified by reading [_vendor/.../layouts/](_vendor/github.com/nicokaiser/hugo-theme-gallery/v4/layouts/) and [layouts/](layouts/).

- **Theme is vendored, not module-fetched.** CMS doesn't need to handle Hugo Module fetching.
- **Images are auto-discovered** from each page bundle ([_vendor/.../partials/gallery.html:23](_vendor/github.com/nicokaiser/hugo-theme-gallery/v4/layouts/partials/gallery.html#L23)). Adding a JPEG to an album folder is sufficient — **no frontmatter update needed for an append.**
- **Display order is alphabetical by filename** (`sort_by: Name`, asc). `Stevie-1.jpg`–`Stevie-10.jpg` would sort `1, 10, 2, 3...` — sequence numbers must be zero-padded once an album crosses 9 photos.
- **Featured-image fallbacks** ([_vendor/.../partials/get-gallery.html:4](_vendor/github.com/nicokaiser/hugo-theme-gallery/v4/layouts/partials/get-gallery.html#L4)):
  1. Exact-match `featured_image:` in frontmatter.
  2. First file matching glob `*feature*` (case-sensitive).
  3. First image alphabetically.
- **Frontmatter fields actually consumed**: `title`, `date`, `description`, `featured_image`, `tags`, `private`, `featured`, `sort_by`, `sort_order`, `theme`, `weight`, `layout`, `story`, `display_date`. The `categories` and `series` taxonomies in [hugo.toml](hugo.toml) are defined but never rendered — CMS can ignore.
- **Two separate "featured" mechanisms**: `featured: true` in album frontmatter promotes that album to the homepage card (newest such album wins). [content/featured/](content/featured/) is a *separate page*, also featured.
- **i18n is monolingual.** The seven [i18n/](i18n/) files are theme cruft; `[languages]` is undefined.
- **Build takes ~41s for the full site.** Run only at "Publish" time.
- **Build failure surface is narrow.** Hard-fails: invalid YAML. Silently-wrong: typo'd `featured_image:`, `private` + `featured` together. CMS form must validate semantics up front.
- **Story pages** use [layouts/_default/story.html](layouts/_default/story.html) and a different shape — out of scope for v1.

**Your image facts (verified):** Nikon Z8 → Lightroom Classic, sRGB, 1600×1067 ~400 KB JPG, EXIF includes `DateTimeOriginal`, no GPS data embedded, `Orientation` flag honored by Hugo's AutoOrient.

---

## Image strategy (the big architectural decision)

**Convert everything to WebP at 1200×800, quality 80.**

**Math:**
- Current: 1600×1067 JPG q85 ≈ 400 KB × 3700 photos = **2.5 GB source** / 2.7 GB built
- After: 1200×800 WebP q80 ≈ 100 KB × 3700 = **~370 MB source** / ~700 MB built (-85%)

**Why this and not the alternatives:**
- **External "originals" store rejected.** Your 1600px JPGs aren't real originals (those are RAW in your Lightroom catalog). Hosting them externally adds ops complexity (R2 credentials, two failure modes, CORS) for a maybe-once-a-year download case. Rare high-res requests get handled by email + LR re-export.
- **WebP-in-repo accepted** despite URL changes (`.jpg` → `.webp`). You said you don't care about broken inbound links and prefer the size win. The few hardcoded `.jpg` references in [content/about.html](content/about.html) (`<img src="/images/Sophie-20240224-1.jpg">`) get rewritten by the conversion script.
- **CMS converts JPG → WebP at import time.** Your Lightroom workflow doesn't change — keep exporting JPG (you said WebP from LR is painful). The CMS handles the format conversion and resizing as part of "Publish." Defense in depth: even if you forget to update the LR preset, repo stays tight.

**One-shot conversion script** (~30 lines `sharp`): walks [content/albums/](content/albums/), [content/featured/](content/featured/), [content/openstudios/](content/openstudios/), and [static/images/](static/images/), converts each `.jpg` to `.webp` at 1200×800 q80, deletes the original `.jpg`, rewrites references in [content/about.html](content/about.html) and any other hardcoded mentions. Runs once before the CMS goes live.

---

## Tool shape

**Local web app.** Runs on your laptop with `npm run cms`. Opens a browser tab pointed at `localhost`, reads/writes the repo directly, shells out to `git`. Bound to `127.0.0.1` only (LAN can't reach it). No accounts, no auth, no deployment, no users but you.

**Stack**: small Node backend (Express/Fastify) + small frontend (SvelteKit, Vite, or vanilla — TBD at build time). ~1000–1500 lines total. Image processing via `sharp`.

**Considered and rejected**: hosted git-CMSes like Decap/Sveltia. They're built around editing markdown bodies, handle Hugo Page Bundles awkwardly, and don't fit a 2.5 GB image-heavy repo. Wrong tool.

---

## UX spec

### Browse view (the home screen)

A scrollable list of all albums:

```
┌─────────────────────────────────────────┐
│ 🔍 Search...                  A B C ... │  ← type-ahead + A–Z jump nav
├─────────────────────────────────────────┤
│ [thumb] acre        5 photos · Dec 2024 │
├─────────────────────────────────────────┤
│ [thumb] addisyn     4 photos · Sep 2024 │
├─────────────────────────────────────────┤
│ [thumb] max-1       7 photos · Jun 2023 │
│ [thumb] max-2       3 photos · Aug 2024 │  ← thumbnail makes "which Max" obvious
└─────────────────────────────────────────┘
```

- Each row: featured-image thumbnail + folder name + photo count + most-recent shoot date.
- A–Z jump nav (iOS Contacts style) for fast navigation through 743 rows.
- Type-ahead search across folder name and `title:`.
- Sort options: alphabetical (default), newest shoot first, oldest shoot first, "no description yet," "fewest photos."
- Lazy-load thumbnails as rows scroll into view.
- Top-level "+ New Album" button.

### New-album flow

1. Click "+ New Album."
2. Drag-and-drop JPGs from a Lightroom export folder.
3. Type the dog's name → live duplicate detection:
   - **Zero matches** → green "creating new album: `acre`."
   - **One match** → modal: "An album for `acre` already exists. Same dog (add to it) or new Acre (suggest `acre-2`)?" with thumbnail of existing.
   - **Multiple matches** (`max-1`, `max-2` already exist) → modal: list each with thumbnail, "Which one is this, or new Max-3?"
   - **Similar names** (`Maya` vs `Maia`) → soft warning, not blocking.
4. Tag auto-fills with `slug(name)` lowercase (e.g. `Max` → tag `max`). Optional "nicknames" field for the rare `["joe", "sloppy joe"]` case.
5. Date auto-fills from EXIF `DateTimeOriginal` of the first photo.
6. Featured image picker: thumbnail grid, click a star to set.
7. Description field (optional, rarely used — not prompted aggressively).
8. Click "Publish" → preview the diff (file list + frontmatter) → second click commits.

### Edit-album flow

1. From browse view, click an album row.
2. View shows: existing photos in display order, current frontmatter (read-only by default), drag-zone for new photos.
3. Drag in new JPGs → tool computes next sequence number, shows the planned filenames, side-by-side preview with existing photos.
4. **Metadata is read-only by default.** A toggle ("Edit metadata") unlocks title/date/tags/featured-image fields. Prevents accidental keystrokes from nuking a 2-year-old album's title.
5. Click "Publish" → preview the diff → second click commits.

### Disambiguation UX

This is the single most important screen — picking the wrong "Max" silently appends shots to the wrong dog's archive. Always show **thumbnails + photo count + most-recent date** for each candidate. Never let the user proceed without explicit confirmation.

### Thumbnails

- CMS generates ~150×150 thumbs on first run, caches in `.cms-cache/thumbs/` (gitignored).
- ~7 MB total cache for 743 albums. Built once, near-instant after.
- Re-generated on demand when a new photo is published.

---

## Safeguards

The 743 existing albums are the project's archive — losing or corrupting them would be a real loss. Once the tool can edit existing albums, it can also break them.

### Repo / data integrity
- **Bind to `127.0.0.1` only.** LAN can't reach the CMS.
- **Refuse to start on a dirty working tree.** Any uncommitted changes from outside the CMS = stop. Surface them, don't bundle.
- **Lockfile** (`.cms.lock`) prevents two browser tabs from writing simultaneously.
- **Atomic writes**: write to a temp folder, then `mv` into place. Crashes never leave half-created albums.
- **Folder-name uniqueness** enforced at every step:
  - Slug format: `^[a-z][a-z0-9-]*$`. Lowercase ASCII only. No spaces, commas, diacritics. (Verified: zero existing folders contain commas or spaces. The `Leo, Olive` pattern is filenames-only.)
  - Existing folders mix capitalization (`acre` vs `Cubby`) — they stay as-is. New folders all lowercase.
  - Live duplicate check while typing.
  - Re-checked atomically at write time.
  - Compared **case-insensitively** (macOS HFS+/APFS is case-insensitive; Hugo lowercases URLs anyway).
- **Image filename uniqueness within an album.** Append computes next sequence number from the highest existing one. Collision = hard fail, not overwrite.
- **Always zero-pad new sequence numbers.** New files always written as `name-YYYYMMDD-01.webp`, `-02.webp`, etc. Existing albums keep their existing pattern (the CMS reads what's there and continues it).
- **One folder = one dog, forever.** Reshoots append to the existing folder. The `-1` / `-2` suffix is reserved exclusively for two *different* dogs sharing a name.
- **Tag = real dog name** (no folder suffix). `Max-1` and `Max-2` both have tag `max` so a search for `max` finds both.
- **No bulk operations in v1.** Each action touches exactly one album folder.
- **Add `.DS_Store` to [.gitignore](.gitignore)** before any of this.

### Git safety
- **Stage and show a diff, don't auto-commit.** Publish button stages + previews; second click commits.
- **Never push automatically.** You push manually when ready.
- **Never run destructive git commands** (`reset --hard`, `clean -f`, `checkout --`). Only `status`, `add`, `commit`.
- **Each album = one commit**, message style matches your existing pattern: `adding Acre`, `adding 3 more photos of Acre`.

### Build safety
- **Form validates semantic constraints up front.** `featured_image:` must reference a real file in the folder; `featured: true` + `private: true` warns; tags lowercased and trimmed.
- **Run `hugo --quiet` before committing as a backstop.** ~41s. Catches YAML parsing edge cases the form missed. Failure → refuse to commit, surface error.
- **Optional `hugo server` preview** on a different port for visual confirmation before publish.

### Image safety
- **Convert JPG → WebP at import time** via `sharp`: 1200×800 max, quality 80, EXIF preserved (`DateTimeOriginal`, `ImageDescription`, `Orientation`).
- **Read date from EXIF `DateTimeOriginal`**, fall back to file mtime, fall back to today.
- **Pre-rotate per Orientation flag and clear it** to avoid Hugo's AutoOrient rotating again.
- **Keep originals** — CMS *copies* JPGs in, doesn't move them. Your Lightroom export folder stays intact.
- **Hugo regenerates resized derivatives at build time** into `resources/` (gitignored). CMS doesn't manage these.

### Existing-album mode specifically
- **Read-only on metadata by default.** Explicit "Edit metadata" toggle to unlock title/date/tags/featured-image fields.
- **Show before/after thumbnails** when appending. Filenames visible. Confirm before write.
- **Big "this album already has N photos" indicator** at the top of the edit view.

---

## Pre-CMS cleanup (one-shot scripts, separate from CMS work)

These run *once*, before or alongside CMS rollout. Each is a small standalone Node/bash script.

1. **WebP conversion** of all 743 existing albums + [content/featured/](content/featured/), [content/openstudios/](content/openstudios/), [static/images/](static/images/). Updates hardcoded `.jpg` references in [content/about.html](content/about.html). One commit: `convert all images to WebP`.
2. **Tag normalization** (lowercase all tags, strip whitespace). One commit: `normalize tag capitalization`.
3. **Add `.DS_Store` to [.gitignore](.gitignore).** Trivial.
4. **Git history shrink** — *after* the WebP conversion settles. Use `git-filter-repo` to drop old `.jpg` blobs from history. `.git` should drop from 3.8 GB → ~500 MB. **Rewrites public history; force-push required; you'll need to re-clone on any other machines.** Do this last, deliberately, on a quiet day.

---

## Out of scope for v1

Named explicitly so we don't drift:
- Renaming or deleting existing albums (URL invalidation).
- Deleting individual images from existing albums.
- Editing static pages: [content/about.html](content/about.html), [content/support.html](content/support.html), [content/story/](content/story/).
- Editing [content/featured/](content/featured/) or [content/openstudios/](content/openstudios/) (different shapes).
- Toggling `featured: true` from the CMS (manual frontmatter edit for now).
- Auto-updating the "775 dogs / 2500 sessions" stat in [content/about.html](content/about.html). Could become `data/stats.json` later.
- Image-level duplicate detection (hash check on import).
- Linking dogs to their stories in [content/story/](content/story/).
- Multi-language content (site is monolingual English).
- Categories / series taxonomies (defined but never rendered).
- Description backfill on the 740 blank descriptions (you may want it later — leave the field, don't prompt aggressively).

---

## Hosting watch-out

GitHub Pages 1 GB soft limit isn't enforced today. After WebP conversion, built site goes from 2.7 GB → ~700 MB — comfortably under. Adding albums extends the runway substantially.

If GitHub ever does enforce or you outgrow it again: **Cloudflare Pages** is the easiest escape hatch — same `git push` model, free tier, no size or bandwidth caps. ~30 minutes of work to migrate when needed.

---

## Critical files

- [hugo.toml](hugo.toml) — site config
- [content/albums/_index.md](content/albums/_index.md) — albums section frontmatter
- [content/albums/acre/index.md](content/albums/acre/index.md) — representative per-album
- [.github/workflows/hugo.yaml](.github/workflows/hugo.yaml) — deploy pipeline
- [layouts/](layouts/) — local theme overrides (CMS reads frontmatter shape; doesn't edit)
- [_vendor/github.com/nicokaiser/hugo-theme-gallery/v4/layouts/](_vendor/github.com/nicokaiser/hugo-theme-gallery/v4/layouts/) — theme source for reference
- [.gitignore](.gitignore) — needs `.DS_Store` and `.cms-cache/` added

## Verification (when we build)

- **New album round-trip**: create one via CMS, run `hugo server`, verify it renders correctly (homepage + album list + album page) with right featured image and tags.
- **Append round-trip**: pick an existing album, drag in 2 new photos, publish, verify new photos appear in correct order with correctly-numbered filenames; rest of album unchanged.
- **WebP conversion round-trip**: run conversion script on a copy of the repo first, build with `hugo`, spot-check 10 albums in browser to confirm visual quality is acceptable.
- **Diff matches hand-made**: `git diff` after a CMS run looks identical to a commit you'd have made by hand.
- **Existing-content invariant**: a new-album CMS run produces zero changes to any other album folder. Verify with `git diff --stat`.
- **Failure modes**: feed it a duplicate album name, duplicate image filename, dirty working tree, malformed frontmatter, oversized image, HEIC file. Confirm refuses cleanly in each case.
