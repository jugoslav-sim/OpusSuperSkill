---
name: opus-super-skill
description: >
  Core engineering discipline for Claude while coding in Jugoslav's projects. Use this skill for ANY
  coding task — writing features, fixing bugs, refactoring, reviewing, or setting up infrastructure —
  in either of his two stacks: (A) React 19 + Vite + Firebase + Express 5 + Dexie + Capacitor + Polar
  on Vercel, or (B) Next.js 16 App Router + Supabase + Drizzle + Tailwind v4 + shadcn/ui + Anthropic SDK.
  Even for "small" changes, consult this skill first — it encodes the verification gates, security
  boundaries, and stack gotchas that prevent the most common regressions.
---

# Opus Super Skill — How to code well in this workspace

These are the highest-leverage rules, ordered roughly by how often ignoring them causes real damage.
The two stack-specific files under `references/` carry the framework gotchas — read the one that
matches the repo you're in **before** writing code:

- `references/stack-vite-firebase.md` — React 19 + Vite 6, Firebase (Auth/Firestore), Express 5, Dexie offline sync, Capacitor, Polar billing, Gemini via Vercel AI SDK.
- `references/stack-nextjs-supabase.md` — Next.js 16 (App Router/RSC), Supabase (Postgres/Auth/Storage), Drizzle, Tailwind v4, shadcn/ui, Anthropic SDK, Tiptap.
- `references/checklists.md` — mechanical pre-ship checklists (new endpoint, new table/collection, new UI view, new env var, AI feature). Walk the matching checklist before declaring any such change done.

You can tell which stack you're in from `package.json` in seconds: `next` → Stack B; `vite` + `firebase` → Stack A.

## 1. Read before you write

Never edit a file you haven't read, and never add a function without first searching for an existing
one that does the job. These codebases already have established patterns (a sync queue, an
entitlement mapper, a Supabase server client factory). Duplicating them creates two sources of truth
that drift apart — the bug shows up weeks later as "works here, broken there". Spend the first
minutes of any task with Grep/Glob finding: the module that owns this concern, the pattern used by
its nearest sibling, and the test file that covers it.

## 2. The definition of done is a passing gate, not a finished edit

A change is not done when the code is written. It is done when:

```
tsc (typecheck) → eslint → vitest (affected tests) → all pass
```

Run these yourself before reporting completion. If a gate fails, fix the root cause — do not
suppress with `@ts-ignore`, `eslint-disable`, or `.skip` unless the user explicitly approves.
If you couldn't run a gate (e.g. no test exists for the changed logic), say so plainly instead of
implying it was verified. For behavior that pure tests can't reach (UI, auth flows), state exactly
what manual step the user should perform to confirm.

## 3. Minimal diff, maximal intent

Change only what the task requires. No drive-by reformatting, no renaming things you happen to
dislike, no "while I'm here" refactors — they bloat review, hide the real change, and create merge
conflicts with in-flight work. If you spot genuinely broken adjacent code, report it as a finding
rather than silently fixing it. The corollary: within the lines you *do* change, match the file's
existing style exactly (naming, import ordering, comment density).

## 4. Types are the spec — never weaken them

- No `any`, no `as unknown as`, no non-null `!` to silence the compiler. If the type is hard to
  satisfy, the design is telling you something.
- Every external boundary gets a Zod schema: API request bodies, webhook payloads, AI structured
  output, environment variables. Parse, don't cast — `schema.parse(input)` gives you a validated
  value *and* the type, so the runtime and compile-time views can't diverge.
- Derive types from the source of truth instead of restating them: `z.infer<typeof schema>`,
  Drizzle's `$inferSelect/$inferInsert`, Supabase generated types. Hand-written duplicates rot.

## 5. Trust boundaries are non-negotiable

Both stacks are client-heavy apps with server-side secrets, which makes this the #1 category of
career-ending bug:

- **Secrets never reach the client.** AI keys (Gemini/Anthropic), Firebase Admin credentials,
  Supabase service-role key, Polar/webhook secrets — server code only. Anything prefixed
  `VITE_` / `NEXT_PUBLIC_` ships in the bundle; treat those prefixes as radioactive for secrets.
- **Every API route/server action authenticates first, then authorizes.** Verify the Firebase ID
  token / Supabase JWT server-side, then check the resource actually belongs to that user. The
  client's word ("here is my userId") is never evidence.
- **Verify webhook signatures** (Polar, Brevo, etc.) before touching the payload. An unverified
  webhook endpoint is a free "grant me a subscription" API.
- **Row-level protection is defense-in-depth, not the only wall.** Firestore rules / Supabase RLS
  must exist AND server code must still scope queries by the authenticated user.
- Validate all user input server-side with Zod even when the client already validated it.

## 6. Design for the failure path first

The happy path is easy; production quality lives in what happens when things fail:

- Every `await` that crosses a network (Firestore, Supabase, AI call, Polar API) can reject —
  decide *at the call site* whether to retry, degrade, or surface, and never swallow the error
  silently.
- AI calls specifically: they time out, return malformed output, and hit rate limits. Always parse
  the output with the Zod schema, and have a defined behavior when parsing fails (retry once, then
  degrade with a user-visible message — never crash and never fabricate a fallback value).
- Make mutations idempotent where re-delivery is possible (webhooks, offline sync queue replays).
  Keying writes on a stable external ID is usually enough.
- User-facing errors get a human message via the app's existing mechanism (sonner toasts in Stack A);
  logs get the real error with context. Don't mix these up in either direction.

## 7. React 19 discipline: state down, events up, effects last

`useEffect` is the tool of last resort, for synchronizing with systems *outside* React (subscriptions,
browser APIs). It is not for derived state (compute during render), not for responding to prop
changes (compute or key-reset), and not for data fetching when the stack provides better tools
(RSC/server components in Stack B, Dexie `useLiveQuery` in Stack A). Most "impossible" bugs in
React apps are effect-ordering bugs that this rule prevents. Keep components small, hoist state no
higher than needed, and give list items real keys (stable IDs, never array index for mutable lists).

## 8. Schema and data migrations are one-way doors

Dexie version upgrades, Drizzle migrations, and Firestore document-shape changes all share a
property: shipped clients / existing rows still have the old shape. Before changing any persisted
schema: write the migration (never edit an applied migration or an existing Dexie version block —
always add a new version), make readers tolerant of both shapes during rollout, and stop before
running anything destructive (drop/delete/rename of populated columns) — that's a decision for the
user, not you.

## 9. Say what you did, honestly and briefly

Lead with the outcome: what changed, whether the gates pass, and any risk you're aware of. If tests
fail, show the failure — don't narrate around it. If you made an assumption (picked a library,
guessed at intended behavior), name it explicitly so it can be corrected cheaply now rather than
expensively later.

## 10. When stuck, widen — don't thrash

If the same fix has failed twice, stop editing. Re-read the error verbatim, form a hypothesis about
the *mechanism*, and verify it with a targeted probe (log, minimal repro, reading the library's
actual types in `node_modules`) before the third attempt. Fixes applied without a confirmed
mechanism are guesses, and stacked guesses corrupt the codebase.

## 11. A new dependency is an architectural decision, not a convenience

Before `npm install`-ing anything: check whether an existing dependency already covers it (both
stacks already carry a validation library, a date story, an icon set, a toast system), whether a
few lines of your own code would do, and what it costs — client bundle size in a PWA/WebView app,
serverless compatibility (no native binaries) on the API side, and a maintenance surface that
outlives the feature. If a new package genuinely earns its place, say so explicitly in your summary
with the reason, so the human can veto cheaply.

## 12. A UI feature isn't the happy-path render

Every view you build or touch ships with all four states — loading, empty, error, and populated —
because users hit the first three constantly and demos never do. Alongside that, the accessibility
floor is non-negotiable: interactive elements are real `<button>`/`<a>`/form controls (keyboard and
screen-reader behavior for free), inputs have labels, images have alt text, focus is visible and
managed on dialogs, and color is never the only signal. Retrofitting this costs 5× writing it in.

## 13. Scale the process to the blast radius

A one-line fix doesn't need a plan document; a change spanning schema + API + UI does. Before any
multi-file change, briefly state the plan — files to touch, order (schema/data layer → server →
client), and what could break — then execute. Sequencing matters: land the layer that others depend
on first so the codebase typechecks at every intermediate step and you always have a working state
to retreat to. When a task turns out bigger than it looked mid-flight, stop and say so rather than
silently expanding scope.
