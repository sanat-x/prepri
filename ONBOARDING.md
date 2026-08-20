# PrepRI — getting set up

You've been added as a collaborator. This walks you from nothing to being able to contribute. Should take about 45 minutes, most of it waiting on installers.

---

## What we're building

An app where Indian students find events run by other students — football tournaments, case competitions, essay competitions, fundraisers, internships, workshops, and socials. Right now these only spread through Instagram stories and WhatsApp groups, so they reach the organizer's existing circle and nobody else. PrepRI puts them in one place, filtered by your interests and your city, with a friends layer so you can see what people you know are going to.

Launching Chennai-first, because the whole thing depends on local density.

Stack: React + Vite, Tailwind, Supabase (Postgres database + auth), deployed on Vercel.

**Important:** the app doesn't exist yet. The repo currently holds the build spec and some conventions — no code. We're generating it with Claude Code in five phases, and you're getting set up before that starts.

---

## What's already done

- GitHub repo: `sanat-x/prepri` (you're a collaborator)
- Supabase project created — one shared database, we both point at it
- Vercel connected to the repo — every push builds automatically and produces a live URL
- Build spec written: `CLAUDE_CODE_PROMPT.md`
- Conventions written: `CLAUDE.md`

## What you need to do

### 1. Install the tools

(You've already got push access to the repo, so there's nothing to accept — start here.)

**Node.js** — nodejs.org, download the LTS version, run the installer.

**Git for Windows** — git-scm.com. Accept the defaults. Claude Code uses this for its Bash tool, and you need git regardless.

**Claude Code** — open PowerShell and run:

```powershell
irm https://claude.ai/install.ps1 | iex
```

Your Claude Pro plan covers it. Note that Pro has lower usage limits than Max, so I'll take the heavy code-generation phases and you take lighter work — no point burning your weekly cap on scaffolding.

**GitHub CLI** (saves you a painful authentication detour):

```powershell
winget install GitHub.cli
```

Then:

```powershell
gh auth login
```

Pick GitHub.com → HTTPS → yes to authenticating git → login with browser. Approve in the browser tab that opens.

### 2. Tell git who you are

```powershell
git config --global user.name "Your Name"
git config --global user.email "your-github-email@example.com"
```

Use the email on your GitHub account, otherwise your commits won't be attributed to you and your contribution history looks empty.

### 3. Clone the repo

```powershell
cd ~\Documents
git clone https://github.com/sanat-x/prepri.git
cd prepri
```

You should see `CLAUDE.md`, `CLAUDE_CODE_PROMPT.md`, and `README.md`. That's everything, for now.

### 4. Create your `.env.local`

I'll send you two values separately — a Supabase project URL and a publishable key. Don't paste them into the repo, Slack, or anywhere public. Create a file called `.env.local` in the `prepri` folder:

```
VITE_SUPABASE_URL=<the URL I sent you>
VITE_SUPABASE_PUBLISHABLE_KEY=<the key I sent you>
```

It's gitignored, so it stays on your machine. We each keep our own copy pointing at the same database.

### 5. Read two files

Before you write anything: `CLAUDE_CODE_PROMPT.md` (the full build spec — schema, features, phases) and `CLAUDE.md` (our working rules).

Then tell me if anything in there looks wrong or missing. Genuinely — I wrote the spec in one pass and it hasn't been reviewed by anyone. Easier to fix now than in Phase 4.

---

## Once the app exists

After I run Phase 1, the repo gets a real project. From then on:

```powershell
git pull           # get my latest work
npm ci             # install dependencies (use ci, NOT install)
npm run dev        # start the local server, opens on localhost:5173
```

**Never run `npm audit fix --force`.** It silently upgrades major versions and desyncs our two machines. I already lost time to this once.

---

## How we work

**Branches, never `main` directly.**

```powershell
git pull
git checkout -b friends-page
# do the work
git add .
git commit -m "Add friend request accept flow"
git push
```

Git prints a link to open a pull request. Vercel comments on it with a preview URL within a minute or so. Send me that link, I review on my phone, you fix anything, we merge.

Keep branches to about a day. Week-old branches conflict with everything.

**Who owns what**, so we're not editing the same files:

- Me — events: `Discover`, `EventDetail`, `CreateEvent`, `EditEvent`
- You — social: `Profile`, `EditProfile`, `Friends`

`lib/queries.js` and `lib/constants.js` are shared. Say something in chat before editing either.

**Database migrations — the one that will actually bite you.**

We share one Supabase database. There's no separate dev copy.

Migration files are just text in the repo. Committing one changes nothing; someone has to paste it into the Supabase SQL Editor by hand. So if you pull my branch and the app starts erroring about a column that doesn't exist, that's why — check `supabase/migrations/` for anything unapplied.

And going the other way: if you run a migration, you're changing my database while I'm working. So tell me first, run it, then tell me to pull. Never write a `DROP` or `DROP COLUMN` without asking.

**Claude Code, scoped.** At the start of a session, tell it what it's allowed to touch:

> We're on branch `friends-page`. Only modify `src/pages/Friends.jsx` and `src/lib/queries.js`.

Left open-ended it'll refactor six files you didn't ask about, and that becomes a merge conflict in code neither of us wrote.

It reads `CLAUDE.md` automatically at the start of every session — that's why the conventions live in the repo rather than in our heads. `CLAUDE_CODE_PROMPT.md` is not auto-loaded; point at it explicitly when it's relevant.

---

## Never

- Commit `.env.local`
- Put a Supabase key starting `sb_secret_` anywhere in this project — it bypasses the database's security rules entirely
- Push to `main` directly
- Run a destructive migration without asking

---

## If you get stuck

`git push` returning a 403 means authentication, not permissions — rerun `gh auth login`. Anything else, message me with the exact error text.
