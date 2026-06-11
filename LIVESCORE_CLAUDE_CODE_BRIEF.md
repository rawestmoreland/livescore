# LocalScore — Claude Code Brief

## Realtime Sports League Tracker (v2 — TanStack Start + PocketBase)

---

## Project context

A lightweight, realtime score and standings tracker for small local sports leagues — bowling alleys, rec soccer, pub darts, youth basketball. No registration required to view scores. Organizers set up a league in minutes and share a public link. Realtime updates push to all viewers during live games.

Intentionally **not** a full league management suite. No payments, no complex registration flows, no deep roster management. The scorekeeper view is a PWA — installable to a phone home screen and optimized for one-handed use.

---

## Tech stack

### Backend — PocketBase

Single Go binary. Built-in realtime SSE subscriptions, auth, REST API, and admin UI. Deploy on a $4–6/mo Hetzner or Fly.io VPS. No Docker required. Collections: `leagues`, `teams`, `games`, `score_events`. A `pb_hooks` JS hook handles score increments server-side.

### Frontend — TanStack Start (React + TypeScript)

Full-stack React framework with SSR. File-based routing via TanStack Router. Use `createServerFn` for data fetching on public SSR routes. PocketBase JS SDK (`pocketbase` npm package) for auth and realtime subscriptions on the client. No special adapter needed — TanStack Start handles SSR + client hydration.

### Server state — TanStack Query

Use `useSuspenseQuery` for initial data load on all routes. Layer PocketBase's realtime subscription inside a `useEffect` to push updates into the query cache via `queryClient.setQueryData`. This keeps SSR data and live updates in sync through a single data layer.

### Styling — Tailwind CSS v4

Utility-first. No component library — keep the bundle lean and the scoreboard UI custom. Add a `manifest.webmanifest` and service worker for PWA installability on the scorekeeper route.

### Deployment

PocketBase binary + TanStack Start Node server on the same VPS. Nginx reverse proxy: port 80/443 → Node (web), `/api/` and `/_/` → PocketBase (port 8090). One-command deploy via a shell script in the repo root.

---

## Realtime pattern

> **Realtime is the core differentiator.** Every public viewer on a live game page must see score changes within ~500ms of entry — no polling, no manual refresh.

The pattern for every realtime-enabled page:

```ts
// 1. Load initial data server-side (SSR)
const loaderData = Route.useLoaderData();

// 2. Seed query cache from loader data
const { data: game } = useSuspenseQuery({
  queryKey: ['game', gameId],
  queryFn: () => pb.collection('games').getOne(gameId),
  initialData: loaderData.game,
});

// 3. Subscribe to realtime updates → push into cache
useEffect(() => {
  pb.collection('games').subscribe(gameId, ({ record }) => {
    queryClient.setQueryData(['game', gameId], record);
  });
  return () => pb.collection('games').unsubscribe(gameId);
}, [gameId]);
```

Also subscribe to the `games` collection (filtered by `league_id`) on the league hub page so a game transitioning to `live` or `final` surfaces without a reload.

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

| File                         | Purpose                                              | Rendering       |
| ---------------------------- | ---------------------------------------------------- | --------------- |
| `routes/index.tsx`           | Product landing + create league CTA                  | SSR             |
| `routes/$slug.tsx`           | Public league hub — live scores, standings, schedule | SSR             |
| `routes/$slug.game.$id.tsx`  | Live game view — realtime score, period, event log   | SSR             |
| `routes/dashboard.tsx`       | Organizer dashboard — manage leagues, games, teams   | CSR, auth-gated |
| `routes/scorekeeper.$id.tsx` | Mobile-first live score entry. PWA. PIN auth.        | CSR             |

> **Note:** SSR only applies to public routes. The dashboard and scorekeeper routes are client-rendered — disable SSR per-route via TanStack Start's route options or wrap in a `ClientOnly` boundary.

---

## Auth model

**Public (no auth):** View any league hub, game scoreboard, standings. SSR-rendered, fully public, OG meta tags for link sharing.

**Organizer (email/password via PocketBase auth):** Create/edit leagues, teams, games. Access `/dashboard`. Store the PocketBase auth token in a cookie (httpOnly) for SSR access. Use a TanStack Router `beforeLoad` guard to redirect unauthenticated users.

**Scorekeeper (6-digit PIN):** A PIN is stored on each `games` record. The scorekeeper route prompts for the PIN, validates against PocketBase, and stores a short-lived token in `sessionStorage`. Cannot access any other route. Screen Wake Lock API should be enabled to prevent phone sleep mid-game.

---

## Project structure

```
localscore/
├── pb/                              # PocketBase
│   ├── pocketbase                   # binary (gitignored)
│   ├── pb_migrations/               # schema as migration files
│   ├── pb_hooks/
│   │   └── scores.pb.js             # score increment + undo hook
│   └── seed.js                      # demo league seed script
├── web/                             # TanStack Start app
│   ├── app/
│   │   ├── routes/
│   │   │   ├── index.tsx            # landing
│   │   │   ├── $slug.tsx            # league hub (SSR)
│   │   │   ├── $slug.game.$id.tsx   # live game (SSR)
│   │   │   ├── dashboard.tsx        # organizer (CSR, auth-gated)
│   │   │   └── scorekeeper.$id.tsx  # score entry (CSR, PWA)
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
│   │   ├── router.tsx
│   │   └── client.tsx
│   ├── public/
│   │   ├── manifest.webmanifest     # PWA manifest
│   │   └── sw.js                    # service worker (scorekeeper)
│   └── app.config.ts                # TanStack Start config
├── nginx.conf
└── README.md
```

---

## Build phases

### Phase 1 — PocketBase schema + seed

Create all collections with correct field types, indexes, and API rules. Write a seed script that creates a demo league ("Riverside 5-a-side") with 4 teams and 6 sample games in various statuses. Verify realtime works with a simple HTML test client before touching the React app.

### Phase 2 — TanStack Start scaffold + PocketBase integration

Init TanStack Start project. Install `pocketbase`, `@tanstack/react-query`. Create the `pb.ts` singleton (handles both server and client environments — check `typeof window`). Write `queryOptions` factories for leagues, games, and standings. Confirm SSR data loading works on the slug route with a static fixture.

### Phase 3 — Public league hub (`$slug.tsx`)

SSR page showing league name/sport, live games (highlighted), standings table, upcoming schedule. Wire up realtime subscription for live game status and score changes. Add OG meta tags (title, description, image) for link sharing previews.

### Phase 4 — Live game view + scorekeeper entry

SSR game view with large score display, period indicator, and event log. Separate `scorekeeper.$id.tsx` (CSR only) with large +/- tap targets (min 56px), period control, undo last event, and "End game" inline confirmation. Enable Screen Wake Lock API. Add PWA manifest so it's installable.

### Phase 5 — Organizer dashboard

Auth-gated `/dashboard` with `beforeLoad` redirect guard. Create league form with slug availability check (debounced PocketBase filter call). Team management. Game scheduler with home/away picker and datetime. Generate and display scorekeeper PIN per game.

### Phase 6 — Polish + deployment

Empty states, error boundaries, loading skeletons. Nginx config. README with step-by-step deploy instructions (buy VPS → install Node + PocketBase → clone repo → run `deploy.sh`). PocketBase backup cron example.

---

## Key implementation notes

### Score increment via PocketBase hook — not direct PATCH

Scorekeeper POSTs a new record to `score_events` (append-only). A `pb_hooks/scores.pb.js` hook on `onRecordAfterCreateRequest` atomically increments the correct score field on the parent `games` record. This prevents race conditions and makes "undo last point" trivial — just delete the last `score_events` record and decrement.

### PocketBase singleton — isomorphic safe

TanStack Start runs code on both server and client. The `pb.ts` singleton must guard against SSE subscriptions being opened server-side. Only call `pb.collection(...).subscribe()` inside `useEffect` (client-only). For server data fetching in `createServerFn`, use a separate server-only PocketBase instance with admin credentials from env vars.

### Query cache as the single source of truth

Don't use `useState` for server data. All game/league data lives in TanStack Query's cache. SSR seeds it via `initialData` in `useSuspenseQuery`. Realtime pushes update it via `queryClient.setQueryData`. Components just read from the cache — they never know if data came from SSR or a realtime push.

### Standings as a PocketBase view collection

Do not maintain standings as a writeable collection. Create a PocketBase view collection that aggregates `games WHERE status = 'final'` grouped by team. Query it like any other collection. It recalculates automatically on each read — no sync jobs needed at this scale.

### Slug uniqueness + availability check

The league slug is the public URL identifier. Add a unique index on `leagues.slug` in PocketBase. The create-league form should debounce-check availability via a lightweight filter API call as the user types.

### Scorekeeper view — mobile-first

The `/scorekeeper` route must work one-handed on a phone. Score buttons must have a minimum 56px tap target. Undo last event must be accessible without scrolling. Avoid modals — use inline confirmation for destructive actions like "End game". Enable the Screen Wake Lock API (`navigator.wakeLock.request('screen')`) on mount to prevent the phone sleeping mid-game.

---

## Out of scope for v1

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
- [ ] The league hub page renders correctly when shared as a link (SSR + OG meta tags populate)
- [ ] The scorekeeper route is installable as a PWA from mobile Safari and Chrome
- [ ] Full app deploys to a fresh VPS following only the README instructions
