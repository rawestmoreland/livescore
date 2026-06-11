# LocalScore — Claude Code Brief

## Realtime Sports League Tracker (v3 — SPA + PocketBase)

---

## Project context

A lightweight, realtime score and standings tracker for small local sports leagues — bowling alleys, rec soccer, pub darts, youth basketball. No registration required to view scores. Organizers set up a league in minutes and share a public link. Realtime updates push to all viewers during live games.

Intentionally **not** a full league management suite. No payments, no complex registration flows, no deep roster management. The scorekeeper view is a PWA — installable to a phone home screen and optimized for one-handed use.

---

## Architecture decision: SPA, no SSR

This app does not need SSR. Every route is either:

- **Realtime-first** — the page is stale the moment it's server-rendered anyway; a PocketBase subscription immediately overwrites it
- **Auth-gated** — organizer dashboard and scorekeeper have no SEO or sharing value
- **Navigated to internally** — users reach game pages from the league hub, not cold links

The only legitimate SSR use case would be OG meta tags on the public league hub (`/[slug]`) for link previews in iMessage/WhatsApp. This is handled with a lightweight meta tag endpoint on the PocketBase server (see below) — not a full SSR framework.

**Use Vite + TanStack Router as a pure SPA.** No TanStack Start, no hydration complexity, no isomorphic PocketBase gotchas. PocketBase's realtime subscriptions are designed exactly for this use case.

---

## Tech stack

### Backend — PocketBase

Single Go binary. Built-in realtime SSE subscriptions, auth, REST API, and admin UI. Deploy on a $4–6/mo Hetzner or Fly.io VPS. No Docker required. Collections: `leagues`, `teams`, `games`, `score_events`. A `pb_hooks` JS hook handles score increments server-side.

### Frontend — Vite + TanStack Router (React + TypeScript)

Pure SPA. File-based routing via TanStack Router. No server-side rendering. PocketBase JS SDK (`pocketbase` npm package) for auth and realtime subscriptions. Built with Vite — fast dev server, simple static output.

### Server state — TanStack Query

Use `useQuery` / `useSuspenseQuery` for all data fetching. Layer PocketBase realtime subscriptions inside `useEffect` to push live updates into the query cache via `queryClient.setQueryData`. Components only ever read from the cache — they don't care if data arrived from an initial fetch or a realtime push.

### Styling — Tailwind CSS v4

Utility-first. No component library — keep the bundle lean and the scoreboard UI custom. Add a `manifest.webmanifest` and service worker for PWA installability on the scorekeeper route.

### OG meta tags (optional, lightweight)

A single PocketBase JS hook serves a minimal HTML shell with populated `<meta>` tags when a crawler `User-Agent` hits `/[slug]`. ~30 lines. Everything else is served as the standard SPA `index.html`. This is the only "server rendering" in the project.

### Deployment

PocketBase binary + Vite static build (`dist/`) on the same VPS. Nginx serves the static files and proxies `/api/` and `/_/` to PocketBase (port 8090). One-command deploy via a shell script in the repo root.

---

## Realtime pattern

> **Realtime is the core differentiator.** Every public viewer on a live game page must see score changes within ~500ms of entry — no polling, no manual refresh.

The pattern for every realtime-enabled component:

```ts
// 1. Initial data fetch via TanStack Query
const { data: game } = useQuery({
  queryKey: ['game', gameId],
  queryFn: () => pb.collection('games').getOne(gameId),
});

// 2. Subscribe to realtime updates → push into cache
useEffect(() => {
  pb.collection('games').subscribe(gameId, ({ record }) => {
    queryClient.setQueryData(['game', gameId], record);
  });
  return () => pb.collection('games').unsubscribe(gameId);
}, [gameId]);
```

Also subscribe to the `games` collection (filtered by `league_id`) on the league hub so a game transitioning to `live` or `final` surfaces without a reload.

---

## Data model

```
leagues
  id, name, sport, slug (unique, URL-safe), logo_url?,
  organizer_user_id, created, updated

teams
  id, league_id (→leagues), name, abbreviation (3 char),
  color_hex, created

games
  id, league_id, home_team_id (→teams), away_team_id (→teams),
  scheduled_at, started_at?, ended_at?, status (scheduled|live|final),
  home_score (int, default 0), away_score (int, default 0),
  period_label? (e.g. "Q3", "2nd Half", "Frame 7"), notes?

score_events            ← append-only log
  id, game_id (→games), team_id (→teams), delta (int),
  recorded_by (user_id), recorded_at

standings               ← computed via PocketBase view collection
  (derived from games WHERE status = 'final')
  team_id, wins, losses, draws, points, games_played
```

---

## Routes

| File                         | Purpose                                                         |
| ---------------------------- | --------------------------------------------------------------- |
| `routes/index.tsx`           | Product landing + create league CTA                             |
| `routes/$slug.tsx`           | Public league hub — live scores, standings, schedule            |
| `routes/$slug.game.$id.tsx`  | Live game view — realtime score, period, event log              |
| `routes/dashboard.tsx`       | Organizer dashboard — manage leagues, games, teams. Auth-gated. |
| `routes/scorekeeper.$id.tsx` | Mobile-first live score entry. PWA. PIN auth.                   |

All routes are client-rendered. No SSR anywhere.

---

## Auth model

**Public (no auth):** View any league hub, game scoreboard, standings.

**Organizer (email/password via PocketBase auth):** Create/edit leagues, teams, games. Access `/dashboard`. Store the PocketBase auth token in `localStorage` (PocketBase SDK handles this automatically). Use a TanStack Router `beforeLoad` guard to redirect unauthenticated users to a login page.

**Scorekeeper (6-digit PIN):** A PIN is stored on each `games` record. The scorekeeper route prompts for the PIN, validates against PocketBase, and stores a short-lived token in `sessionStorage`. Cannot access any other route. Screen Wake Lock API should be enabled to prevent phone sleep mid-game.

---

## Project structure

```
localscore/
├── pb/                              # PocketBase
│   ├── pocketbase                   # binary (gitignored)
│   ├── pb_migrations/               # schema as migration files
│   ├── pb_hooks/
│   │   ├── scores.pb.js             # score increment + undo hook
│   │   └── og-meta.pb.js            # crawler meta tag handler (optional)
│   └── seed.js                      # demo league seed script
├── web/                             # Vite + TanStack Router SPA
│   ├── src/
│   │   ├── routes/
│   │   │   ├── index.tsx            # landing
│   │   │   ├── $slug.tsx            # league hub
│   │   │   ├── $slug.game.$id.tsx   # live game view
│   │   │   ├── dashboard.tsx        # organizer (auth-gated)
│   │   │   └── scorekeeper.$id.tsx  # score entry (PWA)
│   │   ├── lib/
│   │   │   ├── pb.ts                # PocketBase client singleton
│   │   │   ├── queries/             # queryOptions factories per entity
│   │   │   │   ├── leagues.ts
│   │   │   │   ├── games.ts
│   │   │   │   └── standings.ts
│   │   │   └── hooks/
│   │   │       └── useRealtimeGame.ts   # subscribe + cache update
│   │   ├── components/
│   │   │   ├── Scoreboard.tsx
│   │   │   ├── StandingsTable.tsx
│   │   │   ├── GameCard.tsx
│   │   │   └── ScorekeeperPad.tsx
│   │   ├── main.tsx
│   │   └── router.tsx
│   ├── public/
│   │   ├── manifest.webmanifest     # PWA manifest
│   │   └── sw.js                    # service worker (scorekeeper)
│   └── vite.config.ts
├── nginx.conf
└── README.md
```

---

## Build phases

### Phase 1 — PocketBase schema + seed

Create all collections with correct field types, indexes, and API rules. Write a seed script that creates a demo league ("Riverside 5-a-side") with 4 teams and 6 sample games in various statuses. Verify realtime works with a simple standalone HTML test file that subscribes to the `games` collection before touching the React app.

### Phase 2 — Vite + TanStack Router scaffold + PocketBase integration

Init Vite project with React + TypeScript template. Install `@tanstack/react-router`, `@tanstack/react-query`, `pocketbase`. Set up TanStack Router with file-based routing. Create the `pb.ts` client singleton. Write `queryOptions` factories for leagues, games, and standings. Confirm data fetching and realtime subscriptions work end-to-end against the seeded PocketBase instance.

### Phase 3 — Public league hub (`$slug.tsx`)

Page showing league name/sport, live games (highlighted), standings table, upcoming schedule. Wire up realtime subscription so game status and score changes update without a reload. This is the most-viewed route — make it fast and readable on mobile.

### Phase 4 — Live game view + scorekeeper entry

Game view with large score display, period indicator, and event log. Separate `scorekeeper.$id.tsx` with large +/- tap targets (min 56px), period control, undo last event, and "End game" inline confirmation. Enable Screen Wake Lock API (`navigator.wakeLock.request('screen')`) on mount. Add PWA manifest so the route is installable.

### Phase 5 — Organizer dashboard

Auth-gated `/dashboard` with `beforeLoad` redirect guard. Login page. Create league form with slug availability check (debounced PocketBase filter call). Team management. Game scheduler with home/away picker and datetime. Generate and display scorekeeper PIN per game.

### Phase 6 — Polish + deployment

Empty states, error boundaries, loading skeletons. Nginx config. README with step-by-step deploy instructions (buy VPS → install Node + PocketBase → clone repo → run `deploy.sh`). PocketBase backup cron example. Optional: add the OG meta tag hook (`og-meta.pb.js`) for link preview support.

---

## Key implementation notes

### Score increment via PocketBase hook — not direct PATCH

Scorekeeper POSTs a new record to `score_events` (append-only). A `pb_hooks/scores.pb.js` hook on `onRecordAfterCreateRequest` atomically increments the correct score field on the parent `games` record. This prevents race conditions and makes "undo last point" trivial — just delete the last `score_events` record and decrement.

### Query cache as the single source of truth

Don't use `useState` for server data. All game/league data lives in TanStack Query's cache. Initial data arrives via `useQuery`. Realtime pushes update it via `queryClient.setQueryData`. Components just read from the cache — they never know whether data came from a fetch or a realtime event.

### PocketBase singleton — client only

Unlike an SSR setup, there is no isomorphic concern here. The `pb.ts` singleton is a plain module-level instance created once on the client. PocketBase SDK handles auth token persistence in `localStorage` automatically. Subscriptions can be opened anywhere — no `typeof window` guards needed.

### Standings as a PocketBase view collection

Do not maintain standings as a writeable collection. Create a PocketBase view collection that aggregates `games WHERE status = 'final'` grouped by team. Query it like any other collection. It recalculates automatically on each read — no sync jobs needed at this scale.

### Slug uniqueness + availability check

The league slug is the public URL identifier. Add a unique index on `leagues.slug` in PocketBase. The create-league form should debounce-check availability via a lightweight filter API call as the user types.

### Scorekeeper view — mobile-first

Score buttons must have a minimum 56px tap target. Undo last event must be accessible without scrolling. Avoid modals — use inline confirmation for destructive actions like "End game". Enable the Screen Wake Lock API on mount to prevent the phone sleeping mid-game. The route should work reliably on a $200 Android phone on a spotty gym WiFi connection.

### Nginx SPA fallback

The Nginx config must include `try_files $uri $uri/ /index.html` so that deep-linking directly to `/riverside-fc` or `/scorekeeper/abc123` works correctly rather than returning a 404.

---

## Out of scope for v1

- Server-side rendering (SSR)
- React Native / Expo
- Push notifications
- Payment processing
- Player registration flows
- Video streaming
- Player-level statistics
- Multi-sport tournament brackets
- Email / SMS alerts

---

## Success criteria for v1

- [ ] An organizer can create a league, add teams, and schedule games in under 5 minutes
- [ ] A scorekeeper on mobile can increment scores and advance periods with no friction, with the screen staying awake
- [ ] Fans viewing the public scoreboard see score changes within 1 second, no refresh required
- [ ] The scorekeeper route is installable as a PWA from mobile Safari and Chrome
- [ ] Direct links to `/[slug]` and `/scorekeeper/[id]` work correctly (Nginx SPA fallback configured)
- [ ] Full app deploys to a fresh VPS following only the README instructions
