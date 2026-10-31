# App data architecture: one data layer, small screens

**Status (2026-10-30):** built in both apps, except one piece. Both apps have the data layer; Rates & hosting and the Jobs tab are on it and split; every redraw-over-and-over found on the big screens is fixed (table below). Left: splitting the provider's Manage/Service Room modal out of Home (about 720 lines) — the provider's most important flow, so it waits for testing on a device.

## Why

Measured on the live server, 2026-10-30: every provider request took 22–215 ms. The server isn't what makes the apps slow. Two things in the apps are:

1. **Very large single screens.** Provider Home is 4,127 lines and Profile 5,511, each one component with 55–90 pieces of state. In React a component redraws as a whole, so one keystroke redraws the whole screen. On mid-range Android that's the stutter while typing and scrolling.
2. **No shared data layer.** Every screen fetches by hand (`useState` + `useEffect` / `useFocusEffect` + `SkoFyApi`), with its own loading, error and refresh code. The same data (the provider's profile, their skills) is fetched again on every screen and every visit, and the hand-written code is where this week's bugs came from:
   - reloads overwriting what's being typed;
   - parts of a screen arriving at different times;
   - a failed reload replacing the screen with an error;
   - saves done one request after another.

The admin portal already uses TanStack Query and has none of these.

## The design

### 1. One data layer: TanStack Query (`@tanstack/react-query`, as in the admin portal)
- **One `QueryClient`** at the app root (`services/query-client.ts`).
  - Data stays fresh for 30 seconds, so moving between screens doesn't refetch.
  - Kept for 10 minutes after a screen closes.
  - One quiet retry on a network error, never on a 4xx.
  - Knows when the app comes back to the foreground (`AppState`) and when the phone is back online (NetInfo). It refreshes then.
- **Query keys in one place** (`services/queries/keys.ts`). A screen never writes a key by hand, so two screens asking for the same thing share one request and one cached answer.
- **One hook per resource** (`services/queries/*.ts`): `useProviderProfile()`, `useMySkills()`, `useHosting()`…
  - Screens call hooks, never `SkoFyApi` for reads.
  - Writes are `useMutation` hooks. They update or invalidate the keys they change, so every screen showing that data updates at once.

### 2. Forms keep their own drafts
Server data and what's being typed are different things. A form copies the server's values into local draft state once, and only resets them on purpose: after a save, or a Discard. A background refresh updates the server copy but never the draft. That ends the "reload wiped my typing" class of bug by design, instead of guarding each screen.

### 3. Small components that redraw on their own
- A screen file is the layout and the data hooks.
- Each section is its own component in `components/<screen>/`, wrapped in `React.memo` and given only what it shows plus stable callbacks.
- Target: no screen file over ~600 lines.

### 4. Pull-to-refresh and focus
- `usePullRefresh` takes a query's `refetch`.
- Returning to a screen refreshes data only if it's stale (older than 30 s). The query client does that; screens don't hand-write `useFocusEffect` loads.

## Phases (each shippable on its own, verified before the next)

| Phase | Scope | Status |
|---|---|---|
| 1 | Data layer: package, `QueryClient` with app-state and network awareness, keys, first hooks | Built |
| 2 | Rates & hosting on the data layer, split into components | Built |
| 3 | Jobs tab (542 lines) | Built |
| 4 | Provider Home (4,127 lines): its data on hooks, then split section by section | Data on the layer; job cards split out (`JobFeedCard`); redraws fixed. The Manage/Service Room modal is the next piece to split |
| 5 | Provider Profile (5,511 lines): the same | Clocks fixed (availability, skill test); profile and skills read through the shared cache. Not split further |
| 6 | Customer app: the same plan, heaviest screens first | Data layer built (same client, keys, write refresh, sign-out clear); profile, addresses and active jobs shared by Home and the bottom bar; redraws on Home and Track provider fixed |

## Redraws found and fixed (2026-10-30)
Things that redrew a whole big screen over and over, found while splitting:

| Screen | Was | Now |
|---|---|---|
| Home | A clock in the screen's state redrew all of Home every second, for the job cards' countdowns | Each countdown ticks by itself (`Countdown`) |
| Home | Every GPS fix (every 2 s or 5 m) redrew Home and looked the address up over the network | The distances shown follow 30 m moves; the address every 150 m or 2 min. The live location sent to the server is unchanged |
| Profile | A clock redrew all of Profile every second, to hide past availability slots | Once a minute (slots end on the minute) |
| Job details | A clock redrew the job's details every second, for "Remaining time" | `TimeLeft` ticks by itself |
| Home | The jobs feed: loaded twice on opening, redrew Home every 15 s poll even with nothing new, and a failed poll emptied the feed | One shared poll (also serves the Jobs tab), cards rebuilt only when the server's lists change, a failed poll keeps what's shown |
| Home | Every job card was an inline render function, redrawn with any change on Home | `JobFeedCard`, memoised: a card redraws for its own job |
| Home | Profile fetched twice on opening; profile, skills and job alerts fetched again on every visit | Shared with Rates & hosting through the data layer |
| Profile | The skill test's clock redrew all of Profile every second of a test | `TestCountdown` ticks by itself; Profile hears only of a pause or time up |
| Customer Home | A timer moved every provider pin by a random step every 3 s ("simulated" movement) — false locations, and all of Home redrawn | Removed: pins are where the server says |
| Customer Home | Each 15 s refresh replaced the providers, jobs and regular providers with new copies, redrawing Home with nothing new, and looked each map-tapped job's address up again | A refresh with nothing new keeps what's shown (`keepIfSame`); a job's address is looked up once |
| Customer Home | Profile and saved addresses fetched twice on opening, and the bottom bar fetched the profile and jobs a third time | One request each, shared through the data layer |
| Rates & hosting | A provider with every skill got a full card per skill, all drawn at once — thousands of price boxes: slow to open, stuck while scrolling | A list drawn as it scrolls (`FlatList`); each skill one line with its prices until tapped; a search over the skills; skills priced by quote named together in one card |
| Track provider | The vehicle's glide redrew the whole 1,900-line screen 20 times a second | `GlidingProviderMarker` redraws only the marker; standing still draws nothing |

Still redrawing each second, only while running: the call timer in both chats.

## Rules for new code
- No clock or GPS value in a big screen's state: give it to the small component that shows it.
- No `SkoFyApi` reads inside a screen. Add a hook in `services/queries/`.
- No `useFocusEffect` that refetches. The query client handles freshness.
- A form's drafts are local state, never overwritten by a refetch.
- A new section of a big screen goes in its own component file.
