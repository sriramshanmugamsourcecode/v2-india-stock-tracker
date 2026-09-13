# Developer Environment Setup — India Stock Tracker v2

This document covers everything needed on a developer's machine, and every external account/service, to work on this codebase. Read this before [`02-developer-reference.md`](02-developer-reference.md), which explains *how* those services are wired together.

## 1. What this app actually is

The entire application is **one static HTML file** (`index.html`) — vanilla HTML/CSS/JavaScript, no framework, no build step, no bundler. There is nothing to `npm install` and nothing to compile. Editing the file and opening it in a browser (or pushing it to GitHub Pages) *is* the whole development loop.

The only "backend" is [Supabase](https://supabase.com) (hosted Postgres + Auth), reached directly from the browser via its JS client library. Live stock prices come from Yahoo Finance's public (unofficial) chart-data endpoint, reached through a small Cloudflare Worker that exists solely to get around browser CORS restrictions.

## 2. Software to install locally

| Tool | Needed for | Notes |
|---|---|---|
| [Visual Studio Code](https://code.visualstudio.com/) | Editing `index.html` | This is the actual editor this project has been developed in. Any text editor is technically capable of editing a plain HTML file, but VS Code is what's assumed if you want to replicate this project's setup, mainly because of the extension below. |
| [Git](https://git-scm.com/) | Version control, pushing to GitHub | Standard install for your OS. |
| A modern browser (Chrome, Edge, or Firefox) | Previewing the app | You can open `index.html` directly from disk (`file://…`) for most UI work. Supabase calls work fine over `file://`; nothing in this app requires a local dev server. |
| [Python 3](https://www.python.org/) | **Optional** — only for scripts in `tooling/` (e.g. `sync_universe.py`, the Nifty-500 universe reconciliation tool) | These scripts use only the Python standard library — no `pip install` of anything is required. |
| [Claude Code extension for VS Code](https://marketplace.visualstudio.com/) (search "Claude Code" in the Extensions panel) | Not required to run the app, but this entire codebase has in practice been built and maintained through it | This project has been developed by pairing with Claude inside VS Code via this extension: it reads/edits `index.html` directly, runs `git` and SQL-diagnostic commands from an integrated terminal, and uses its **Artifact** tool to publish a clickable HTML mockup of a new feature for review *before* any code is written, per this project's standing "discuss before acting, mock before building" workflow. Requires a [Claude](https://claude.ai) account with API/Code access — a new developer isn't required to use it to run the app, but continuing development the way this project has actually been run assumes it. |

**Explicitly *not* needed:**
- **Node.js / npm** — there is no JavaScript build step. The two third-party JS libraries the app uses (Supabase client, SheetJS) are loaded straight from a CDN via `<script src="…">` tags in `index.html` — nothing is installed locally or bundled.
- **A local web server** — not required for development, though nothing stops you from running one (e.g. `python -m http.server`) if you prefer serving over `http://` to `file://`.
- **Any broker/trading API credentials** (e.g. Zerodha Kite Connect) — trade import works by the user manually exporting a `.xlsx` file from their broker's website and uploading it into the app, which parses it client-side. No API key, token, or paid broker-API plan is involved anywhere in this project.

## 3. Accounts and external services

**No paid licenses are required anywhere in this project.** Every service below has a free tier that comfortably covers a personal, single-user (or small-family) portfolio tracker.

| Service | What it's for | Plan |
|---|---|---|
| [GitHub](https://github.com) account | Hosts the git repo; GitHub Pages serves the live app straight from the repo (no separate hosting account needed) | Free — note that free-tier GitHub Pages requires the repo to be **public** |
| [Supabase](https://supabase.com) account | Postgres database + user authentication, managed through **Supabase Studio** — the web dashboard at `app.supabase.com` for your project | Free tier is sufficient at this scale. Every schema change, RLS policy, and one-off data fix this project has ever needed was run through Studio's **SQL Editor** (`Table Editor` and `Authentication` tabs are also used, but SQL Editor is where the real work happens) — there is no separate migration-runner or ORM in this project. |
| [Cloudflare](https://cloudflare.com) account | Hosts the tiny CORS-proxy Worker used to fetch Yahoo Finance data from the browser | Free tier (Cloudflare Workers) |

None of these require a credit card for the free tier as of writing, though that can change — check each provider's current terms before assuming.

## 4. Libraries loaded via CDN (no local install, no license cost)

These are pulled live from a CDN by `<script>`/`<link>` tags at the top of `index.html` — there is nothing to install or vendor locally:

- **`@supabase/supabase-js@2`** (via jsdelivr) — official Supabase JS client. MIT licensed.
- **`xlsx@0.18.5`** (SheetJS, via jsdelivr) — parses the uploaded broker `.xlsx` files client-side. Apache-2.0 licensed at this version.
- **Google Fonts** (`DM Mono`, `Syne`, `DM Sans`) — loaded from `fonts.googleapis.com`. Free, Open Font License.

## 5. Where credentials actually live

This repo (`repo/`) is **public** on GitHub (required for free-tier Pages), so it must never contain real secrets — only non-secret identifiers (project URLs, refs, the anon key, which is safe by design and already sits inside `index.html` itself).

Actual credentials — your Supabase account login, a GitHub token, your Cloudflare login, the app's own Supabase Auth email/password — live in a **local-only file, one directory above this repo**: `PROJECT_SECRETS_LOCAL_ONLY.md`. It is deliberately kept *outside* `repo/` so it's physically impossible to `git add` it by accident. This mirrors how the original v1 project's `Starting Point/PROJECT_CONTEXT.md` already worked (real credentials, never committed) — same pattern, continued for v2.

If that file doesn't exist on a machine you're setting up fresh, recreate it from this structure and fill in the real values yourself — never paste real secrets into a Claude Code (or any AI) conversation to have it filled in for you.

The Supabase CLI has also been linked locally at points (`supabase/.temp/linked-project.json`, sitting alongside `repo/`, confirms it points at the v2 project) — but it is **not** the actual migration workflow this project uses; every schema change has been run by hand through Supabase Studio's SQL Editor (see §3 above). The CLI link is incidental, not something you need to actively use.

## 6. A known, accepted risk: Yahoo Finance access

Live prices, 52-week highs, and momentum-screening data all come from Yahoo Finance's public chart-data JSON endpoint (`query1.finance.yahoo.com`). This is **not** an official, licensed, or paid Yahoo API — it's the same publicly-reachable endpoint Yahoo's own website uses, accessed the way many hobby finance projects do. There is no SLA, no guarantee it keeps working, and Yahoo could change or block it at any time without notice. This is a deliberate, accepted trade-off for a personal project, not an oversight — just be aware of it if the app suddenly stops fetching prices one day.

## 7. Getting the repo

```bash
git clone https://github.com/sriramshanmugamsourcecode/v2-india-stock-tracker.git
cd v2-india-stock-tracker
```

Two git remotes matter here (see [`02-developer-reference.md`](02-developer-reference.md) for the full explanation):

- `v2` → `v2-india-stock-tracker` — this is the one you push to. GitHub Pages serves the app from this repo's `main` branch.
- `origin` → the original v1 tracker repo. **Push is intentionally disabled** to this remote in this project's history — v1 is a separate, frozen predecessor and must never be modified from this codebase.

Work on the `v2-india-stock-tracker` branch (or a feature branch off it), and when a change is ready to go live, push to **both** `v2 <branch>` and `v2 HEAD:main`, since Pages only serves `main`.

## 8. Material that exists but isn't in this repo

A few things relevant to this project live only on the original development machine, not in git anywhere. They're documented (not moved) here — see [`02-developer-reference.md`](02-developer-reference.md) for full detail on each:

- **`tooling/`** (sibling folder to `repo/`) — `sync_universe.py` (Nifty-500 universe reconciliation script) and a parked, never-deployed Cloudflare Worker experiment. Not version-controlled.
- **`Starting Point/`** — the original seed materials from before this v2 project began: v1's `index.html` and `PROJECT_CONTEXT.md` (real v1 credentials — historical only, ignore for v2 work), `nifty500_universe.sql`, and a `Backtest/` folder with the original momentum-strategy backtest notebooks.
- **`References/ValuendMomentumEverywhere.pdf`** — an academic paper referenced during a past feature discussion.

None of this is required to develop or run the current v2 app day-to-day — `index.html` alone is sufficient — but if you're trying to fully reconstruct this project's history, these are the pieces to go find.

## 9. That's it

There is no `.env` file, no build config, no dependency lockfile to manage. Once you have Git, a text editor, and accounts with GitHub/Supabase/Cloudflare, you can open `index.html`, make a change, and refresh a browser tab to see it.
