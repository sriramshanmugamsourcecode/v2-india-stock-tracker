# New User Onboarding — Self-Hosting India Stock Tracker v2

*This walks a stranger with no prior involvement through standing up their own, fully independent copy of this app: their own Supabase project, their own GitHub Pages site, their own Cloudflare Worker. Nothing here connects to or depends on the original author's accounts or data. If you just want to *use* an already-hosted instance someone else runs, you don't need any of this — just the URL and a login.*

Estimated time: 20–30 minutes, most of it waiting on account creation and DNS/deploy propagation, not active work.

## Before you start

Read [`01-developer-environment-setup.md`](01-developer-environment-setup.md) for what you need installed and which accounts to create (GitHub, Supabase, Cloudflare — all free tier). Come back here once you have all three.

## Step 1 — Fork the repository

On GitHub, fork (or otherwise copy) the app's repository into your own account. It must be a **public** repo for GitHub Pages' free tier to serve it. The repo is one file that matters: `index.html`. Everything in `docs/` is documentation, not app code.

## Step 2 — Create your own Supabase project

1. At [supabase.com](https://supabase.com), create a new project. Note its **Project URL** and **anon/publishable key** (Project Settings → API) — you'll need both shortly. These are safe to use in client-side code; they're protected by Row Level Security, not by secrecy.
2. Open **SQL Editor** in your new project and run the full schema from [`03-ai-rebuild-spec.md`](03-ai-rebuild-spec.md) §3 — every `create table` / `alter table` / `create policy` statement, in order. This creates all 9 tables with the correct Row Level Security already in place.
3. Create your login: **Authentication → Users → Add user**, enter an email and password. There is no signup screen in the app itself — this is the only way to create an account, by design (this app is meant for you and maybe a few people you trust, not the public).

## Step 3 — Deploy your own CORS proxy (Cloudflare Worker)

The app fetches Yahoo Finance data client-side, but Yahoo doesn't send the CORS headers browsers require for that — so a small proxy sits in between. The original proxy's exact source was never preserved (a known gap in this project — see [`01-developer-environment-setup.md`](01-developer-environment-setup.md) §5), but its behavior is simple enough to reproduce exactly. Here's a working reference implementation:

```js
export default {
  async fetch(request) {
    const url = new URL(request.url);
    const target = url.searchParams.get('url');

    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'GET, OPTIONS',
          'Access-Control-Allow-Headers': '*',
        },
      });
    }

    if (!target) {
      return new Response('Missing ?url= query parameter', { status: 400 });
    }

    try {
      const upstream = await fetch(target, {
        headers: { 'User-Agent': 'Mozilla/5.0' },
      });
      const body = await upstream.arrayBuffer();
      return new Response(body, {
        status: upstream.status,
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Content-Type': upstream.headers.get('Content-Type') || 'application/json',
        },
      });
    } catch (e) {
      return new Response('Proxy fetch failed: ' + e.message, { status: 502 });
    }
  },
};
```

1. In the Cloudflare dashboard, create a new Worker, paste this in, and deploy it. Note the Worker's URL (looks like `https://<name>.<your-subdomain>.workers.dev`).
2. **Save this file somewhere version-controlled this time** (e.g. `tooling/worker/proxy.js` in your fork) — losing track of this exact source is the one concrete gap this documentation project found in the original setup. Don't repeat it.

## Step 4 — Point the app at your own services

In your forked `index.html`, find and update these three lines (currently around line 2134–2139):

```js
const SUPA_URL = 'https://YOUR-PROJECT-REF.supabase.co';       // from Step 2
const SUPA_KEY = 'YOUR-ANON-PUBLISHABLE-KEY';                   // from Step 2
const PROXY    = 'https://YOUR-WORKER-URL.workers.dev/?url=';   // from Step 3
```

Leave `YF_BASE` alone — it's the Yahoo endpoint itself, not something you're hosting.

Commit and push this change to your fork's `main` branch.

## Step 5 — Turn on GitHub Pages

In your fork's repo settings → **Pages** → Source: **Deploy from a branch**, branch `main`, folder `/ (root)`. Confirm a `.nojekyll` file exists at the repo root (it should already be there from the fork) — without it, GitHub's Jekyll processing can interfere with a plain HTML file. Give it a couple of minutes; your app will be live at `https://<your-github-username>.github.io/<repo-name>`.

## Step 6 — Seed the stock universe

The Discovery screener and Rank History need a `universe` table populated with tickers to screen. Run [`universe_seed_v2.sql`](universe_seed_v2.sql) in your Supabase SQL Editor — a real export of the live v2 `universe` table (725 rows) taken 2026-09-12, current as of this writing. (The older `Starting Point/nifty500_universe.sql` from the original project is superseded by this and shouldn't be used for a new setup — it predates several renames and additions.)

Note: that seed file carries forward one known data quirk from the live table rather than silently fixing it — the `CPPLUS` row is stored as `'NSE:CPPLUS'` (with the exchange prefix), unlike every other row. See the comment at the top of the seed file if you want to correct it.

Alternatively, skip this step and add tickers by hand through the app's own **App Maintenance → Universe** panel once you're logged in, growing the list as needed — it doesn't have to be complete on day one.

## Step 7 — Log in and start using it

Open your GitHub Pages URL, sign in with the email/password you created in Step 2. From here:

- **Trade Log → Add Trade** to manually enter a holding, or **Import Kite Trades** if you export a `.xlsx` from Zerodha (or adapt the parser if you use a different broker — see [`04-business-logic.md`](04-business-logic.md) §2 for the exact expected format).
- Everything else — Portfolio, WoW Tracker, Discovery, Rank History, Rebalance — becomes useful once you have at least one trade logged and (for Discovery/Rank History) a populated `universe`.

For what each feature actually does and how its numbers are calculated, see [`04-business-logic.md`](04-business-logic.md) — that document assumes the app is already running and focuses entirely on behavior, not setup.

## If something doesn't work

- **Blank page / console errors about Supabase**: double-check Step 4's three constants are exactly right, no trailing spaces, and that you ran *all* of Step 2's SQL (a missing table shows up as errors specific to whichever feature touches it — this exact failure mode happened once in the original project's own database, see the `rank_status` note in [`03-ai-rebuild-spec.md`](03-ai-rebuild-spec.md) §3).
- **Prices never load**: check your Worker is actually deployed and reachable by visiting its URL directly with a `?url=` pointing at a known Yahoo endpoint; check the `PROXY` constant matches it exactly, trailing `?url=` included.
- **Can't sign in**: confirm the user exists under Supabase → Authentication → Users, and that email confirmation isn't blocking login (Supabase's default settings sometimes require email confirmation — you can disable that requirement in Authentication → Settings for a personal/trusted-users project).
