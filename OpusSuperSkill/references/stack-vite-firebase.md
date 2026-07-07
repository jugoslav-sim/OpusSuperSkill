# Stack A — React 19 + Vite 6 + Firebase + Express 5 + Dexie + Capacitor + Polar

Offline-first PWA (meals app pattern): Dexie (IndexedDB) is the local store for meals and syncs to
Firestore; settings/weight are online-only Firestore. One Express app serves both local dev and
Vercel serverless. Gemini runs server-side via the Vercel AI SDK. Polar handles billing.

## Architecture invariants

- **Single API app, two entry points.** All routes live in `apiApp.ts`; `server.ts` mounts it for
  local dev, `api/index.ts` wraps it for Vercel serverless. Never add a route anywhere else — a
  route added only to `server.ts` works locally and 404s in production, which is the most
  expensive-to-notice bug this architecture allows.
- **Vercel serverless is stateless and can cold-start.** No in-memory caches that matter, no
  `setInterval`, no local file writes, no state shared between requests. Initialize Firebase Admin
  idempotently (`getApps().length ? getApp() : initializeApp(...)`) — module scope re-runs on cold
  start but must survive warm reuse.
- **Data ownership is split and must stay split.** Meals: Dexie is what the UI reads/writes;
  Firestore is the sync target. Settings/weight: Firestore directly, online-only. Don't "helpfully"
  read meals from Firestore in the UI or cache settings in Dexie — each violation breaks either
  offline mode or the sync invariants.

## Express 5

- Express 5 propagates rejected promises from async handlers to the error middleware automatically —
  no `next(err)` boilerplate and no wrapper libs like `express-async-handler`. Rely on the central
  error middleware; keep try/catch only where you add value (mapping to a specific status/message).
- Auth middleware pattern: extract the Bearer token, `admin.auth().verifyIdToken(token)`, attach the
  decoded uid to the request, and scope every Firestore query by that uid. Any route that skips this
  middleware must be deliberately public (webhooks with signature verification, health checks).
- Keep `helmet` and `cors` config intact; when adding a route that needs different CORS, scope the
  exception to that route rather than loosening the global policy.

## Firebase

- **Client SDK vs Admin SDK never mix.** Client (`firebase/*`) in React code; Admin
  (`firebase-admin`) in server code only. Importing Admin into anything Vite bundles leaks
  credentials and breaks the build in confusing ways.
- Firestore security rules must mirror what the API enforces (user can only touch
  `users/{uid}/...` where `request.auth.uid == uid`). When you add a collection, update the rules in
  the same change — a collection without rules is publicly readable or fully locked depending on the
  default, and both are wrong.
- Firestore has no server-computed queries across collections — design documents around read
  patterns. Prefer subcollections under the user doc for user-owned data.
- Timestamps: store Firestore `Timestamp`/server timestamps, convert at the boundary; Dexie stores
  plain numbers/ISO strings. Pick one representation per layer and convert explicitly in the sync
  code, or equality checks in sync diffing silently fail.

## Dexie 4 + sync queue

- **Never edit an existing `db.version(n).stores(...)` block.** Shipped clients have already run
  it. Schema changes = add `version(n+1)` with an `.upgrade()` that migrates in place. Removing an
  index is fine; renaming a table is a migration, not a rename.
- Reads in components go through `useLiveQuery` (from `dexie-react-hooks`) so the UI updates when
  sync writes land — not through one-shot `db.table.toArray()` in effects.
- Sync mutations must be idempotent and replay-safe: queue entries keyed by a client-generated
  stable ID (uuid on the meal), so a retry after a network failure upserts instead of duplicating.
  Conflict policy is last-write-wins on `updatedAt` unless the code says otherwise — keep it
  consistent, don't invent per-field merging in one spot.
- Test sync logic with Vitest against fake queue states (offline burst, replay after failure,
  concurrent edit) — this is exactly the "pure logic" the test suite exists for.

## Capacitor + PWA

- Everything runs inside a WebView on Android/iOS. Guard browser APIs that behave differently there
  (file inputs, notifications, safe-area insets); use Capacitor plugins through a thin wrapper so
  web builds fall back gracefully.
- API base URL differs between web (`/api` same-origin) and Capacitor (absolute URL to the deployed
  API). Route all fetches through the existing API client that handles this — never hardcode a URL
  in a component.
- `vite-plugin-pwa`: bumping cached assets is handled by the plugin, but be deliberate about
  `workbox` runtime caching for `/api` — API responses should generally be NetworkOnly/NetworkFirst;
  a cached API response fights the Dexie offline layer.

## Tailwind (CDN — pending plan 026 migration)

Tailwind currently loads from CDN, so there is no build-time purge and **no custom config**: stick
to stock utility classes; `@apply`, custom theme tokens, and plugins don't exist yet. Don't start
the build-time migration as a side effect of another task — it's plan 026, a deliberate change.

## AI (Vercel AI SDK → Gemini)

- Gemini calls are server-side only, inside the Express app. The `GOOGLE_*` key must never appear
  in client code or `VITE_`-prefixed env.
- Use `generateObject` with a Zod schema for structured output (meal analysis etc.). On schema
  parse failure: retry once, then return a typed error the client renders as a friendly toast —
  never a fabricated meal.
- Model choice: Gemini 2.5 Flash for routine extraction/classification, 2.5 Pro only where quality
  measurably needs it — Pro is slower and pricier per call.
- Bound user input size before sending to the model, and treat model output as untrusted user
  input (validate, don't `dangerouslySetInnerHTML`).

## Polar billing

- Webhooks are the source of truth for entitlements, and they arrive out of order and repeated:
  handle idempotently (key on event ID), verify the webhook signature before parsing, and map
  subscription state → entitlements in the one existing mapping module (it has Vitest coverage —
  extend the tests when you extend the mapping).
- Never trust client-asserted plan/tier; the API reads entitlements from what webhooks persisted.

## Gates & deploy

- CI runs ESLint + `tsc` + Vitest; run all three locally before declaring done.
- Deploy is Vercel: Vite builds the static client, esbuild bundles the server. If you add a server
  dependency, confirm it's serverless-compatible (no native binaries, no filesystem assumptions).
- Env vars exist in three places: local `.env`, Vercel project settings, and (for client values)
  the `VITE_` prefix. New env var = document it and add it to Vercel, or production breaks on the
  next deploy.
