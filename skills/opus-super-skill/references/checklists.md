# Pre-ship checklists

Walk the checklist matching your change *before* declaring it done. These exist because each item
is a real bug class that slips past a green typecheck. Items marked (A) apply to the Vite/Firebase
stack, (B) to the Next.js/Supabase stack, unmarked to both.

## New API endpoint / Server Action

- [ ] Authenticates: verifies Firebase ID token (A) / Supabase `getUser()` (B) — or is deliberately
      public with a written reason (webhook w/ signature check, health check).
- [ ] Authorizes: confirms the resource belongs to the authenticated user, not just that *a* user is
      logged in.
- [ ] Input parsed with a Zod schema; unknown/oversized input rejected with a 4xx, not a crash.
- [ ] Failure paths return correct status codes and a safe message (no stack traces, no internal IDs
      the client shouldn't know).
- [ ] (A) Route added in `apiApp.ts` (not `server.ts`) so it exists in both dev and Vercel.
- [ ] (B) Mutating action revalidates the paths/tags the user sees next.
- [ ] Idempotent if it can be retried/re-delivered (webhooks, sync replays, double-clicked submit).
- [ ] A Vitest covers the interesting logic (extract it into a pure function if it's tangled in the
      handler).

## New table / collection / persisted shape

- [ ] Access policy shipped in the same change: Firestore rules (A) / RLS policies in the same
      migration (B). No table exists without one, even "temporarily".
- [ ] Migration is additive: new Dexie `version(n+1)` with upgrade fn (A) / new generated Drizzle
      migration, applied migrations untouched (B).
- [ ] Readers tolerate both old and new shapes during rollout (shipped clients still write old
      shapes).
- [ ] Types derived from the schema, not hand-restated.
- [ ] Ownership column/path present and every query scopes by it (uid path in Firestore, `userId`
      WHERE clause in Drizzle).
- [ ] Nothing destructive (drop/rename/delete of populated data) without explicit user sign-off.

## New UI view / component

- [ ] All four states implemented: loading, empty (with a helpful next action, not a blank div),
      error (with retry where sensible), populated.
- [ ] Interactive elements are semantic (`button`, `a`, labeled inputs); dialog focus is trapped and
      restored; works with keyboard only.
- [ ] List keys are stable IDs; lists that can be long are paginated or virtualized.
- [ ] No fetching in `useEffect` where the stack has a better tool (RSC (B), `useLiveQuery` (A)).
- [ ] Mutation UX: the submit control disables/spins during flight; failure surfaces via the app's
      toast/error pattern; no double-submit window.
- [ ] (B) `"use client"` is at the lowest node that needs it.
- [ ] Styling uses the design tokens / stock utilities the repo actually supports (A: CDN Tailwind =
      stock classes only; B: `@theme` tokens + shadcn variables).

## New environment variable

- [ ] Named with the right visibility prefix — `VITE_`/`NEXT_PUBLIC_` **only** if it may ship to
      browsers; anything secret gets no prefix.
- [ ] Added everywhere it must exist: local `.env`, Vercel project settings (all needed
      environments), CI if tests read it.
- [ ] Validated at startup (Zod env schema or explicit check) so a missing value fails loudly at
      boot, not deep in a request.
- [ ] Documented (README/env example file) with what it's for and where its value comes from.

## New / changed AI feature

- [ ] Call is server-side; the key never appears in client code or public-prefixed env.
- [ ] Correct key regime (B): per-user key for user-facing assistant work, system key for system
      jobs — verified, not assumed.
- [ ] Structured output parsed with Zod; on parse failure the behavior is defined (bounded retry →
      typed error → friendly message), never a fabricated value.
- [ ] Model matches the job's economics: cheap/fast model (Gemini Flash / Haiku) for bulk or
      routine work; premium (Gemini Pro / Opus) only where quality is the product — cost flagged if
      you upgraded a path.
- [ ] User input bounded (length/size) before it reaches the prompt; model output treated as
      untrusted (sanitized before render, validated before persistence).
- [ ] Timeout/429 handling with backoff; long generations stream instead of buffering toward the
      function limit.

## Before saying "done" (every change)

- [ ] `tsc`, ESLint, and affected Vitest suites run by you and passing — or the gap named
      explicitly.
- [ ] Diff reviewed once end-to-end: no leftover debug logs, commented-out code, stray files, or
      accidental formatting churn.
- [ ] Summary states what changed, what was verified how, assumptions made, and any follow-up the
      human should decide on.
