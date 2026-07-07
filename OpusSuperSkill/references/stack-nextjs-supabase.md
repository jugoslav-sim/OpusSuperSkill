# Stack B — Next.js 16 (App Router) + Supabase + Drizzle + Tailwind v4 + shadcn/ui + Anthropic SDK

Editorial/writing platform pattern: RSC-first Next.js on Vercel, Supabase for Postgres/Auth/Storage,
Drizzle as the server-side ORM with RLS as defense-in-depth, Tiptap editor, Anthropic SDK with
per-user keys (writer assistant) and a system key (aggregation), Brevo email, AdSense.

## Server/client boundary (the #1 source of bugs in this stack)

- Components are **Server Components by default**. Add `"use client"` only when the component needs
  state, effects, event handlers, or browser APIs — and push the directive as far down the tree as
  possible so the RSC shell stays server-rendered.
- **Drizzle, the Supabase service client, and the Anthropic SDK are server-only.** Import them only
  in Server Components, Route Handlers, and Server Actions. Add `import "server-only"` to modules
  that must never be bundled client-side — it turns a silent secret leak into a build error.
- Data flows down as serializable props (no functions, no Dates without care, no class instances
  across the boundary). Mutations flow up via Server Actions or Route Handlers, never by exposing
  a DB client to the browser.
- Server Actions are public HTTP endpoints regardless of where they're defined. Every action must
  authenticate (get the Supabase user) and authorize (check ownership) internally — proximity to a
  protected page protects nothing.

## Supabase Auth (SSR)

- Use `@supabase/ssr` with the established two-client pattern: a browser client for Client
  Components and a per-request server client (cookie-bound) for RSC/actions/route handlers. Never
  cache a server client in module scope — it carries a specific user's cookies.
- **On the server, trust `getUser()` (or `getClaims()`), not `getSession()`** — `getSession()` reads
  the cookie without verifying the JWT signature; `getUser()` validates against Supabase. Auth
  decisions based on `getSession()` server-side are spoofable.
- Middleware/proxy only refreshes the session token; it is not the authorization layer. Real checks
  live next to the data access.

## Drizzle + Postgres + RLS

- Schema lives in the Drizzle schema files; changes go through `drizzle-kit generate` → review the
  generated SQL → migrate. Never edit an applied migration; never let `drizzle-kit push` near
  production data.
- Derive row types from the schema (`typeof table.$inferSelect`) instead of hand-writing
  interfaces.
- Drizzle connects with credentials that (typically) bypass RLS — so **every query must scope by
  the authenticated user in the WHERE clause**. RLS policies are the second wall for anything that
  talks to Postgres via the Supabase clients; when adding a table, add its RLS policies in the same
  migration (a table without policies + anon key access = fully exposed or fully broken).
- Multi-step writes (e.g. create post + revision + notification) go in `db.transaction()`.
  Use the `supabase:supabase-postgres-best-practices` skill for query/index questions.

## Next.js 16 specifics

- `params`/`searchParams`/`cookies()`/`headers()` are async — `await` them. Route Handlers and
  pages that read them become dynamic; be intentional about what's static vs dynamic vs cached
  (`use cache` / cacheLife / cacheTag per the `vercel:next-cache-components` skill).
- After a mutation, revalidate what the user will look at next (`revalidatePath`/`updateTag`) —
  a Server Action that writes but doesn't revalidate produces "saved but UI shows stale data" bugs.
- Don't fetch in `useEffect` what an RSC can fetch on the server; don't add SWR/React Query where a
  server component + revalidation already covers it.

## Tailwind v4 + shadcn/ui

- Tailwind v4 is **CSS-first**: theme tokens live in `@theme` inside the global CSS, not in
  `tailwind.config.js`. Don't create a JS config out of v3 habit; extend the `@theme` block.
- shadcn components are vendored into `components/ui/` — they're yours to edit, but keep edits
  minimal so future `shadcn add` diffs stay readable. New UI: check for an existing shadcn
  primitive (and the `shadcn` skill) before hand-rolling; compose Radix primitives rather than
  reimplementing focus/keyboard/ARIA behavior.
- Style via the design tokens (semantic CSS variables like `bg-background`, `text-muted-foreground`),
  not raw palette values — that's what keeps dark mode and theming coherent.

## Anthropic SDK

- Two key regimes, never confused: **per-user keys** (writer assistant — the user's own key,
  stored encrypted, used only for that user's requests) and the **system key** (aggregation jobs).
  A per-user feature silently falling back to the system key is a billing and abuse bug.
- Model routing is deliberate: Opus 4.8 (`claude-opus-4-8`) for editorial writing/summaries where
  quality is the product; Haiku 4.5 (`claude-haiku-4-5-20251001`) for bulk scoring where volume
  dominates. Don't upgrade a bulk path to Opus for a marginal quality win without flagging cost.
- All calls server-side (route handlers/actions/jobs); stream long generations to the client;
  validate structured outputs with Zod; handle 429/overloaded with bounded retry + backoff. Consult
  the `claude-api` skill for current API parameters rather than memory.
- Treat model output rendered into Tiptap as untrusted content — sanitize/parse into Tiptap JSON,
  never inject raw HTML.

## Tiptap

- The document's source of truth is Tiptap JSON (store JSON, render HTML derivatively). Server-side
  rendering/sanitization of that JSON is the safe path to public pages.
- Extensions must be identical between the editor instance and any server-side renderer, or
  content silently drops nodes.

## Email (Brevo) & Ads (AdSense)

- Brevo sends go through the existing server-side mail module with typed template params; webhook
  handlers verify authenticity and are idempotent. Transactional vs marketing streams stay separate
  (compliance).
- AdSense scripts load via the approved Next.js script strategy (`next/script`, appropriate
  loading strategy) and must not be added inside RSC payload experiments or block LCP; ad slots get
  reserved dimensions to avoid CLS.

## Gates & deploy

- Gates: `tsc`, ESLint, and whatever test runner the repo uses — run before declaring done. Verify
  RLS-relevant changes with the Supabase advisors (`get_advisors`) when the MCP is available.
- Vercel deploy: Server Actions and route handlers run on Fluid Compute (Node), 300s default limit;
  long AI jobs should stream or move to background patterns rather than approach the limit.
- New env vars go into Vercel project settings and `.env.local`; only `NEXT_PUBLIC_` values are
  client-visible — Supabase URL + publishable key are public by design, everything else is not.
