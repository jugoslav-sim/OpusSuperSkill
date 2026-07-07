# OpusSuperSkill

Core engineering discipline for Claude (Opus and up) when coding in Jugoslav's projects, plus
stack-specific gotcha sheets for the two usual tech stacks.

## Contents

The skill lives at `skills/opus-super-skill/`:

- `SKILL.md` — the 13 core instructions (always loaded when the skill triggers)
- `references/stack-vite-firebase.md` — React 19 + Vite 6 + Firebase + Express 5 + Dexie + Capacitor + Polar on Vercel
- `references/stack-nextjs-supabase.md` — Next.js 16 App Router + Supabase + Drizzle + Tailwind v4 + shadcn/ui + Anthropic SDK
- `references/checklists.md` — mechanical pre-ship checklists (new endpoint, new table/collection, new UI view, new env var, AI feature, "before saying done")

Claude reads only the reference file matching the repo it's working in (detected via `package.json`).

## Install

### With the `npx skills` CLI (recommended)

```
npx skills add jugoslav-sim/OpusSuperSkill
```

Add `-a claude-code` to target Claude Code specifically, or `-g` to install into your
user-level skills so it applies to every project.

### Manually

Copy the `skills/opus-super-skill/` folder into the skills directory of the project where you want
it active:

```
<project>/.claude/skills/opus-super-skill/
```

or into your user-level skills so it applies to every project:

```
~/.claude/skills/opus-super-skill/        (Windows: %USERPROFILE%\.claude\skills\opus-super-skill\)
```

Either way it becomes available as `/opus-super-skill` and auto-triggers on coding tasks per the
description in `SKILL.md`.

## Design notes

- Instructions are ordered by blast radius: security/trust boundaries and verification gates first,
  style last.
- The core file stays stack-agnostic so it survives stack churn; anything framework-version-specific
  lives in `references/` where it can be updated independently.
