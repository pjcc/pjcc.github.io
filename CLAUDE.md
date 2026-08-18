# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`pjcc.github.io` is a static personal site published by GitHub Pages at the custom domain in `CNAME` (`piers.qa`). There is no build system, no package manager, no test suite, and no CI - the files in the repo root are served verbatim.

## Working in this repo

- **Deploy = push.** GitHub Pages serves `master` (the default branch is `master`, not `main`). Any commit pushed to `origin/master` is live within a minute or two.
- **Preview locally** by opening the HTML file directly in a browser, or `python -m http.server 8000` from the repo root if a real origin is needed (the dashboard's `fetch` calls require `http://`, not `file://`).
- **Formatting** is Prettier via the VS Code extension (`.vscode/settings.json` sets it as the default formatter). No config file, so Prettier defaults apply. There is no CLI formatter installed - do not add one without being asked.
- Paths inside HTML are **relative** (`style.css`, `assets/favicon.png`) but the SEO metadata (`og:url`, `canonical`, `sitemap.xml`, `robots.txt`) is **absolute against `https://piers.qa/`**. Changing the domain means updating all of those together.

## Structure

- `index.html` - the entire landing page: markup, JSON-LD `Person` schema, SEO/OG/Twitter meta, and an inline dark-mode script. Self-contained apart from `style.css`
- `style.css` - light theme on `body`, dark theme via a `body.dark` class. The toggle script adds/removes that class; there are no CSS custom properties, so a new colour must be added in both blocks
- `db_dboard.html` - a standalone dashboard (see below), not linked from `index.html`
- `assets/projects/` - project card icons: the real `audioflip.ico`, plus two SVGs drawn for the tray app and the extension, which ship no icon of their own. The three sites use their own favicon emoji inline instead
- `assets/` - favicon only. `archive/assets/` holds superseded icons; treat it as dead storage
- `sitemap.xml` currently lists only `/`. New public pages should be added there

### Dark mode contract

The toggle in `index.html` is the source of truth for theming: it reads `localStorage.theme`, falls back to `prefers-color-scheme`, and keeps `aria-pressed` plus `aria-label` in sync on every change. Accessibility attributes on that button have been deliberately fixed in past commits - preserve them when editing.

## Projects section

Six cards below the links in `index.html`, each linking to a `pjcc` repo. Two things about it are load-bearing:

**Dates are baked in *and* fetched.** Each card carries `<time datetime="<full ISO>">` with the real commit date, so the cards are correct with JS off or when the GitHub API is rate-limited (60 req/hour per IP unauthenticated, and the page spends one per card). On load it fetches `/repos/pjcc/<repo>/commits?per_page=1` per card and overwrites the text; failures are swallowed so the baked date survives. The repo name comes from `data-repo`, so adding a project is markup-only.

**The markup order is newest-first and must stay that way.** Sorting runs twice: once immediately off the baked dates - a no-op while the markup is already ordered, which is what stops the cards visibly reshuffling on load - and once more after `Promise.allSettled`, rather than once per fetch. `datetime` holds a full timestamp rather than a date so repos pushed on the same day order by time of day instead of tying.

When updating the baked dates, keep them in newest-first order in the file; the no-JS fallback is the only thing that order serves.

## db_dboard.html (status dashboard)

A single self-contained page that scrapes Downdetector status pages and renders a grid of service health dots. Everything - config, logic, markup - lives in this one file. Key points before editing:

- **Tailwind comes from the CDN** (`cdn.tailwindcss.com`) and the page uses a dark slate palette independent of `style.css`. This page does not participate in the site's dark-mode toggle
- `ALL_SERVICES` (near the top of the `<script>`) is the config: `{ id, cat }` where `id` is the Downdetector slug used to build `https://downdetector.com/status/<id>/` and `cat` is one of `social`, `finance`, `gaming`, `telecom`, `work` (the category filter is driven by these values). Adding a service means adding one entry - nothing else
- **CORS is worked around with a public proxy**, `PROXY = "https://api.allorigins.win/get?url="`. Status is inferred by string-matching the returned HTML for phrases like `"possible problems"`. This is inherently fragile: a Downdetector markup change or a proxy outage breaks detection, and the correct failure mode is severity `3` (`SYNC ERROR`), not a false `OPERATIONAL`
- **Severity is numeric throughout**: `0` operational, `1` possible issues, `2` incident active, `3` sync error. `STATUS_NAMES` and the colour maps (`bg-ok` / `bg-warn` / `bg-down` / `bg-sync-error`) are keyed on it - keep the four in step
- Fetches are batched (10 at a time) over the first `visibleLimit` services, each with a 10s `AbortController` timeout, on a 60s `setInterval` countdown. Raising `visibleLimit` raises proxy load proportionally
- `render()` re-derives the grid from `masterData` on every filter/sort/search change; state changes should mutate `masterData` and call `render()` rather than patching DOM directly (the exception is the two pinned AWS/Cloudflare dots, which are updated by element id)

## Conventions

- Two-space indent, double-quoted attributes and JS strings, semicolons - match the existing files
- British spelling in user-facing copy ("specialising")
- Hyphens, not em dashes, in page copy
