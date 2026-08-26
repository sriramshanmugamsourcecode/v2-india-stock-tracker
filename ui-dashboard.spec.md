# UI / Dashboard Spec

Covers: overall app shell, tab structure, and conventions that span multiple features. See [[portfolio]], [[wow-tracker]], [[discovery-screener]] for each tab's own content.

## 1. Shell

Single-page app, no router — one `index.html`, tab visibility toggled by `switchTab()` show/hiding `<div>`s. Two top-level states: `#login-screen` (email/password form) and `#main-app` (everything else), switched by `db.auth.onAuthStateChange`. No signup UI — accounts are provisioned outside the app.

Header: app title, a live "last updated" timestamp (cycles through `loading...` → `fetching live prices...` → `updated <time>` / `error — see below`), the signed-in user's email, a manual Refresh button, Sign Out.

Four tabs, in this fixed order: **Portfolio | Trade Log | WoW Tracker | ⬡ Discovery**. Discovery is visually distinct — the whole tab goes edge-to-edge (negative margin, `#tab-screener { margin: 0 -16px }`) rather than sitting inside the app's normal max-width container, because it needs the extra horizontal room for its sidebar-filters + wide-table layout.

## 2. Visual style

Dark theme only — no light mode, no theme toggle. Fixed palette (`:root` CSS variables): near-black background (`#0d0f14`), green accent (`#4ade80`/`#22c55e`) for positive/primary actions, red (`#f87171`) for negative/danger, amber (`#fbbf24`) for warnings/near-threshold states, blue (`#60a5fa`) for neutral figures. Three font families loaded from Google Fonts: Syne (display/headings), DM Sans (body), DM Mono (numbers, tickers, timestamps, code-like labels) — numeric/technical values are consistently set in the mono face across every tab.

Responsive down to 600px (stacked header, smaller table padding/font) but not designed mobile-first — tables scroll horizontally rather than reflowing, which is workable but not optimized for phone use.

## 3. Conventions that repeat across tabs

- **Sortable columns**: click a header to sort by it, click again to reverse. Every sortable table in the app (Holdings, Trade Log, each WoW week, Discovery results) implements this independently rather than through a shared component — a real amount of duplicated logic across `sortData`/`wowSortRows`/`scrSort`.
- **Color coding is consistent**: green = positive/pass/above-floor, red = negative/fail/below-floor, amber = warning/near-threshold, muted grey = no-data/neutral. This holds across P&L figures, exit signals, fundamental-indicator dots, and 52W-high proximity coloring.
- **`data-tip` attribute**: a lightweight custom tooltip system (CSS `::after` on hover, no JS) used throughout for explaining what a column/badge/threshold means without cluttering the table itself.
- **Modals**: Import Trades, Manual Trade Entry, and the WoW stock-detail chart all use the same overlay pattern (`position:fixed` full-screen dim, click-outside-to-close, a translateY+opacity transition). Not componentized — each modal's HTML/CSS/JS is separately hand-written.
- **Loading states**: a consistent spinner+label pattern (`.loading` / `.spinner`) rather than skeleton screens; the Portfolio tab specifically renders once immediately with placeholders, then re-renders when live prices arrive, so the page never blocks on the slowest network call.

## 4. Versioning

Single source of truth: `const APP_VERSION = 'v4.20260601.6'` near the top of the `<script>` block, format `v<phase>.<YYYYMMDD>.<build>`. Synced to both `document.title` and the footer text at the very end of the script, so bumping the constant is the only step needed to update both display locations — there is no build tooling that stamps this automatically; it's hand-incremented per session.

## 5. Deployment

No CI/CD. The historical workflow: edit `index.html` locally → paste into GitHub's web file editor → commit directly on `main` → GitHub Pages redeploys automatically within ~2 minutes (`.nojekyll` present, no Actions workflow needed). **This project now has a real local git clone** (`repo/`, cloned 2026-08-26) — going forward, commit-and-push from here instead of the copy-paste-into-GitHub-web-UI flow; nothing else about the deploy mechanism changes since Pages still just watches `main`.
