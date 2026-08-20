# PrepRI

Event discovery app for Indian students. Students post and find events run by other students — tournaments, case competitions, fundraisers, internships, workshops, socials. Launching Chennai-first.

Built by two collaborators, one on macOS and one on Windows. Full build spec is in `CLAUDE_CODE_PROMPT.md` — read it before starting new work.

## Commands

```
npm ci          install dependencies (use this, not npm install)
npm run dev     local dev server
npm run build   must pass before any commit
```

Never run `npm audit fix --force`. It silently upgrades major versions and desyncs the two machines.

## Architecture rules

- **All database calls go in `src/lib/queries.js`.** No component imports the Supabase client directly. If a component needs data, add a named function to `queries.js` and call that.
- Categories, interests, and cities live in `src/lib/constants.js` and must match the database check constraints exactly. Changing one means changing both.
- JavaScript, not TypeScript. React 18, Vite, Tailwind, React Router v6, Supabase.

## Design tokens

<!-- TODO: replace with real values from the Claude Design mockup before Phase 2 -->

Use the tokens defined in `tailwind.config.js`. Do not introduce new colors, fonts, or spacing values ad hoc — if something needs a value that isn't a token, ask first.

Reusable classes (`.btn-primary`, `.card`, `.chip`, `.input-field`) are in `src/index.css`. Prefer them over repeating long Tailwind strings.

## Database migrations — read this before touching the schema

Both collaborators share **one** Supabase project. There is no separate dev database.

That means running a migration changes the other person's environment immediately, while they may be mid-work. So:

1. Migration files go in `supabase/migrations/`, numbered in order.
2. Committing a migration does **not** apply it. Someone has to paste it into the Supabase SQL Editor by hand.
3. Whoever writes a migration: announce it to the other person before running it, then run it, then tell them to pull.
4. Never write a destructive migration (`DROP`, `ALTER ... DROP COLUMN`) without asking the other person first.

If the app throws errors about a column or table that doesn't exist, the cause is almost always an unapplied migration. Pull, then check `supabase/migrations/` for anything not yet run.

## Git workflow

- Never commit directly to `main`.
- Branch per feature: `git checkout -b event-detail`
- `git pull` before starting any new work.
- Push, open a PR, let the other person review the Vercel preview URL, then merge.
- Keep branches short-lived — a day, not a week.

## Ownership

To avoid merge conflicts, work is split by area:

- **Sanat** — events: `Discover`, `EventDetail`, `CreateEvent`, `EditEvent`
- **[friend]** — social: `Profile`, `EditProfile`, `Friends`

`lib/queries.js` and `lib/constants.js` are shared. Say so in chat before editing either.

## Never

- Commit `.env.local`
- Put a Supabase secret key (`sb_secret_`) anywhere in this project — it bypasses Row Level Security. The publishable key (`sb_publishable_`) is the only key this app uses.
- Add analytics, payments, or auth providers that aren't in the build spec
- Refactor files outside the scope of the current task
