# Harbour — Session Handoff

Context document for continuing work on the **Harbour** app. Read this fully before touching anything.

## What and where

- Harbour is a personal daily-tracking app (evening log, weekly "One Thing", evidence journal, morning/evening checklists, habits, metrics, Notion Home List).
- **The app does NOT live in this git repo.** This repo is an unrelated Gatsby portfolio. Harbour lives entirely in **Val Town**, val `brenwildt/Harbour`, edited via the Val Town MCP connector (`mcp__Val_Town__*` tools: `list_files`, `read_file`, `replace_in_file`, `fetch_val_endpoint`, etc.).
- Live endpoint: `https://brenwildt--20a5c902554e11f1a855ee650bb23af1.web.val.run`
- Deploys are instant — editing a file on the val's `main` branch is deployment.

## File map (val `brenwildt/Harbour`)

| File | Role |
|---|---|
| `index.ts` (HTTP) | Hono server: HTML shell, serves the `.tsx` files as `text/babel`, all `/api/*` routes |
| `app.tsx` | App shell: boot data fetching, loading/error states, device frame + tweaks panel wiring |
| `harbour.tsx` | All UI (~1350 lines): theme tokens, atoms (TallyRow, HabitHeatmap, Tick, EnergyMeter, Sparkline), sections (Masthead, OneThing, HowWasToday, Evidence, Checklist, Habits), screens (Today, HabitsScreen, MetricsScreen, HomeScreen), `HarbourToday` root with tab switching + all fetch handlers |
| `db/api.ts` | SQLite read/write functions (std/sqlite via an `exec(sql, args)` wrapper — MUST use object-form binding) |
| `db/init.ts` | Schema init — **never reviewed; read it first** to confirm unique indexes backing the `ON CONFLICT` clauses |
| `db/notion.ts` | Fetches unchecked to-dos from Notion Home List page (`NOTION_TOKEN` env var), returns `{title, category}` |
| `ios-frame.tsx`, `tweaks-panel.tsx` | Cosmetic dev harness (device frame, design tweaks) — not reviewed, leave alone |

Client architecture: no bundler. The HTML shell loads React 18.3.1 **development** UMD builds + `@babel/standalone`, and the four `.tsx` files as `text/babel` scripts transpiled in the browser. Components are shared via `Object.assign(window, ...)`. Scripts must load in order: ios-frame, tweaks-panel, harbour, app.

## Data model (SQLite tables — all exist, no migrations needed)

- `checklist_items(id, session[morning|evening], section, title, detail, sort_order)`
- `checklist_completions(item_id, date, done)` — unique(item_id, date)
- `daily_logs(date, session, energy, note, gratitude, let_go, saved_at)` — unique(date, session)
- `habits(id, name, note, sort_order, active)` — delete is soft (`active=0`)
- `habit_completions(habit_id, date, done)` — unique(habit_id, date)
- `one_thing(id, title, category, week_start, created_at, completed, completed_at)`
- `evidence(id, date, title, detail, logged_at, sort_order)`

All dates are `YYYY-MM-DD` strings. Week starts Sunday.

## API routes (all working, verified 200)

CRUD for checklist items/completions, daily logs, habits, habit-completions (`PUT /api/habit-completions/:habitId/:date` body `{done}`; `GET /api/habit-completions/:habitId?days=28`), one-thing (`GET ?week_start=`, `POST`, `PUT /:id/complete`), evidence, `GET /api/metrics?days=30` → `{days, energy, habits, oneThing}`, `GET /api/notion/home-list` → `[{title, category}]`.

## State: what's already built and live

Four phases shipped and verified (endpoint 200s + clean esbuild transpile of harbour.tsx):

1. **Habits inline on Today** — tap today's cell in `TallyRow` to toggle (`onToggleToday`), edit-mode CRUD mirroring the Checklist pattern; handlers `toggleHabitToday/addHabit/updateHabit/removeHabit` in `HarbourToday`, all optimistic.
2. **Tab navigation** — `HarbourToday` switches its scroll body on `tab` (today/habits/metrics/home); masthead+TabBar persist; `ScreenHeader` shared header component.
3. **Metrics** — `db/api.ts`: `getEnergyHistory`, `getHabitHistory` (computes count/currentStreak/longestStreak server-side), `getOneThingHistory`; route `GET /api/metrics`; `MetricsScreen` with Sparkline energy trend, per-habit streak stats, One Thing history. Lazy-fetches on mount.
4. **Home + Habits tab** — `HomeScreen` renders the Notion list grouped by category (verified: 19 real items). Habits tab shows a 4-week `HabitHeatmap` (28 cells, 7/row, today tappable); `Habits` takes a `wide` prop; `app.tsx` builds both `days` (7) and `history` (28) per habit from one fetch; both stay in sync on toggle.

## Approved fix list (user-approved; implement in this order)

### P0-1: UTC day-boundary bug (most important)
Everything is keyed to UTC dates (`toISOString().slice(0,10)` client-side, `date('now')` in SQLite). User is US Eastern → the app's "day" flips at 8pm EDT, corrupting evening logs/habit ticks/streaks at exactly the time the app is used.
Fix: (a) `app.tsx` `todayStr()`/`currentWeekStart()` and the habit `mk()` day-array builder must use **local** date components (`getFullYear/getMonth/getDate`, zero-padded), not `toISOString`; (b) routes/queries that anchor on `date('now')` (`getHabitCompletions`, `getEnergyHistory`, `getHabitHistory`, metrics route, habit-completions route) accept an optional `today=YYYY-MM-DD` param passed from the client (validate `/^\d{4}-\d{2}-\d{2}$/`), used as `date(?, '-N days')`; `currentStreak` must anchor on that passed date (parse as `T00:00:00Z` and step with UTC methods for consistency); (c) `MetricsScreen` gets a `date` prop (from `data.date`) and sends it.

### P0-2: Masthead is hardcoded
`Masthead` in harbour.tsx literally renders "Tuesday · evening" / "Mar 4". Render real local weekday, session by hour (<12 morning, <17 afternoon, else evening), and `toLocaleDateString("en-US", {month:"short", day:"numeric"})`.

### P0-4: Silent write failures
No client fetch checks `res.ok`. Add a module-level `api(url, opts)` helper in harbour.tsx that throws on non-2xx and parses JSON; convert all ~15 handlers in `HarbourToday`; wrap each in try/catch reporting to a `flash` state (auto-clearing ~4s banner rendered near the top; root div needs `position:relative`). `saveLog` must not `setSaved(true)` on failure.

### P1-5: Production React + drop client Babel
Shell loads dev React + 3MB Babel and transpiles ~65KB JSX per page view on a phone. Fix: import `npm:@babel/standalone@7.29.0` **server-side** in index.ts; `serveCompiled(filename)` transpiles once per isolate (presets `[["typescript", {isTSX:true, allExtensions:true}], "react"]`), caches in a Map, serves `text/javascript`. HTML shell: React/ReactDOM → `.production.min.js`, remove the Babel script tag, plain `<script src="/x.js">` tags in the same order. Verify each `/x.js` route returns compiled JS (no `<`/JSX in output) and `/` loads.

### P1-6: Boot waterfall + N+1
`app.tsx` boot does 6 parallel fetches then one fetch **per habit**. Add `getAllHabitCompletions(days, today)` (single query, all habits) and a `GET /api/bootstrap?date=&weekStart=` route assembling everything Today needs server-side via Promise.all (items, completions, logs morning+evening, habits, all habit completions, oneThing, evidence). Rewrite `app.tsx` `load()` to one fetch + client-side shaping (group completions by habit_id).

### P1-7: Notion cache
`getHomeListItems()` hits the paginated Notion API uncached on every open (Home tab AND One Thing form). Add module-level `{at, items}` cache with ~60s TTL in notion.ts + `invalidateHomeListCache()`.

### P0-3: Actionable Home list (after the plumbing above)
(a) Include block `id` in `HomeListItem`. (b) `checkHomeListItem(blockId)` in notion.ts: `PATCH https://api.notion.com/v1/blocks/{id}` body `{to_do:{checked:true}}`, then invalidate cache. (c) Route `POST /api/notion/home-list/:blockId/check`. (d) `HomeScreen`: checkbox per item (optimistically remove from list on success — list only shows unchecked), plus a per-item "focus →" action that calls the existing `handleSetOneThing(title, category)` and switches to the today tab (pass a callback from `HarbourToday`).

### P2 sweep
- Double-submit on inline edits (Checklist + Habits editors): Enter fires commitEdit and the input's onBlur fires it again → duplicate PUT. Guard with a ref reset in `startEdit`.
- `updateChecklistItem`/`updateHabit`/`updateEvidence` with `{}` build `UPDATE … SET WHERE` (SQL error). Early-return on empty updates.
- Sparkline positions points evenly instead of by date — position x by day offset within the window (needs the anchor date, available after P0-1).
- Remove dead `showSundayPrompt` (computed in app.tsx, set in one handler, never rendered).

## Verification recipe

- After each server change: `fetch_val_endpoint` the affected route (expect 200 + sane JSON). Note: the tool percent-encodes `?` in `pathname` — test query-param routes by their defaults or check behavior accordingly.
- After harbour.tsx changes: read the full file (Val Town read_file → returns JSON with `content`), write to a temp file, run `npx --yes esbuild@0.21.5 file.tsx --outfile=out.js` — must exit 0.
- Don't write test rows into real data without cleaning up; the user is actively using the app.
- `exec()` in db/api.ts exists because std/sqlite silently drops positionally-passed args — always go through it.

## Known constraints

- The sandbox cannot reach the val's public URL directly (proxy 403) — use `fetch_val_endpoint`.
- No browser automation available; final visual check is the user tapping through on their phone.
- Do not create a PR / push app code to this git repo — the app is Val Town-only. This file is the only intended repo artifact.
