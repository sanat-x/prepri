# Build Spec: PrepRI — student event discovery app

The app is called **PrepRI**. Use that name everywhere it's user-visible: the logo in the nav, the page title, the landing page, the README, and the `name` field in `package.json` (as `prepri`). Do not substitute another name.

You are building a web app from scratch. Read this entire spec before writing any code. Work in phases, and stop after each phase so I can test before you continue.

---

## 0. Context

**What the app is:** Indian students post and discover events run by other students — football tournaments, case competitions, essay competitions, fundraisers, internships, workshops, and social events. Today these live on Instagram stories and WhatsApp groups, so they only reach the organizer's existing network. This app puts them in one searchable, personalized place.

**Who uses it:** High school and college students in India, primarily on mobile browsers. Launching hyperlocal in Chennai first, so city filtering matters from day one.

**Two user roles, same account type:** every user can both attend and organize. There is no separate "organizer" account.

**Constraints:**
- I am a high school student. Explain non-obvious decisions in comments and in your messages to me.
- Prioritize working code over clever code. I need to be able to read and modify this myself.
- No feature I didn't ask for. If you think something is missing, tell me — don't just build it.

---

## 1. Stack

- **React 18** + **Vite** (JavaScript, not TypeScript)
- **React Router v6** for routing
- **Supabase** for Postgres database, auth, storage, and realtime
- **Tailwind CSS** for styling
- **date-fns** for date formatting
- Deployment target is **Vercel** — make sure the build works with zero config

Use `@supabase/supabase-js` v2. Do not use any Firebase packages.

---

## 2. Database schema (Postgres via Supabase)

Write these as SQL migration files in `supabase/migrations/`, numbered and ordered. I will paste them into the Supabase SQL editor. Every table gets Row Level Security enabled — no exceptions.

### `profiles`
Mirrors `auth.users`. Created automatically by trigger on signup.

| column | type | notes |
|---|---|---|
| `id` | `uuid` | PK, references `auth.users(id)` on delete cascade |
| `display_name` | `text` | not null |
| `bio` | `text` | default `''` |
| `city` | `text` | nullable |
| `interests` | `text[]` | default `'{}'` |
| `avatar_url` | `text` | nullable |
| `created_at` | `timestamptz` | default `now()` |

Add a `handle_new_user()` trigger function on `auth.users` insert that creates the matching profile row, reading `display_name` from `raw_user_meta_data`. This is important — do not create profiles from client code, because a failed insert would leave an auth user with no profile.

### `events`

| column | type | notes |
|---|---|---|
| `id` | `uuid` | PK, default `gen_random_uuid()` |
| `title` | `text` | not null, check length between 3 and 100 |
| `description` | `text` | not null, check length ≤ 2000 |
| `category` | `text` | not null, check it's in the allowed list (see §3) |
| `starts_at` | `timestamptz` | not null |
| `ends_at` | `timestamptz` | nullable |
| `venue` | `text` | nullable — the specific place, e.g. "KFI school ground" |
| `city` | `text` | not null |
| `tags` | `text[]` | default `'{}'` |
| `max_attendees` | `int` | nullable, check > 0 |
| `cover_url` | `text` | nullable |
| `organizer_id` | `uuid` | references `profiles(id)` on delete cascade, not null |
| `created_at` | `timestamptz` | default `now()` |

Indexes: on `starts_at`, on `city`, on `category`, and a GIN index on `tags` so array-overlap queries are fast.

### `rsvps`
Join table. This replaces stuffing an attendees array on the event.

| column | type | notes |
|---|---|---|
| `event_id` | `uuid` | references `events(id)` on delete cascade |
| `user_id` | `uuid` | references `profiles(id)` on delete cascade |
| `created_at` | `timestamptz` | default `now()` |

Primary key is the composite `(event_id, user_id)` — this makes double-RSVP impossible at the database level rather than relying on client checks.

### `friend_requests`

| column | type | notes |
|---|---|---|
| `id` | `uuid` | PK, default `gen_random_uuid()` |
| `requester_id` | `uuid` | references `profiles(id)` on delete cascade |
| `addressee_id` | `uuid` | references `profiles(id)` on delete cascade |
| `status` | `text` | `'pending'` / `'accepted'` / `'declined'`, default `'pending'` |
| `created_at` | `timestamptz` | default `now()` |

Unique constraint on `(requester_id, addressee_id)`. Check constraint preventing `requester_id = addressee_id`.

### `friendships`
Accepted friendships only. Store each pair **once**, not twice.

| column | type | notes |
|---|---|---|
| `user_a` | `uuid` | references `profiles(id)` on delete cascade |
| `user_b` | `uuid` | references `profiles(id)` on delete cascade |
| `created_at` | `timestamptz` | default `now()` |

Primary key `(user_a, user_b)` plus a check constraint `user_a < user_b`. This ordering trick means the pair (X, Y) and (Y, X) can't both exist — one row per friendship, no sync bugs. Write a helper SQL function `are_friends(uuid, uuid)` that normalizes the order before looking up, and a `get_friends(uuid)` function that returns profile rows regardless of which column the user sits in.

Also write an `accept_friend_request(request_id uuid)` Postgres function that, in a single transaction, marks the request accepted and inserts the friendship row with correctly ordered ids. Doing this server-side means it can't half-complete.

---

## 3. Shared constants

Put these in `src/lib/constants.js` and use them everywhere — the DB check constraints must match this list exactly.

**Categories** (id, label, emoji): `tournament` Tournament 🏆, `competition` Competition ⚡, `fundraiser` Fundraiser 💛, `internship` Internship 💼, `workshop` Workshop 🛠️, `social` Social 🎉, `conference` Conference 🎤, `meetup` Meetup ☕

**Interests:** Sports, Tech, Business, Debate, Music, Arts, Film, Literature, Coding, Finance, Entrepreneurship, Volunteering, Politics, Science, Photography, Gaming, Fashion, Food, Travel

**Cities:** Chennai, Bengaluru, Mumbai, Delhi, Hyderabad, Kolkata, Pune, Ahmedabad, Jaipur, Other

---

## 4. Row Level Security policies

Write these carefully and explain each one in a SQL comment. The rules I want:

- **profiles** — any authenticated user can read any profile. A user can only update their own. Inserts happen only via the signup trigger.
- **events** — any authenticated user can read all events. A user can insert an event only when `organizer_id = auth.uid()`. Only the organizer can update or delete their own event.
- **rsvps** — a user can read all RSVPs (needed for attendee counts and for seeing which friends are going). A user can insert or delete a row only where `user_id = auth.uid()`. This is the key one: it means a user can RSVP for themselves and nobody else, enforced by the database rather than by trusting the frontend.
- **friend_requests** — readable only by the requester or the addressee. Insert allowed only when `requester_id = auth.uid()`. Update (to accept/decline) allowed only by the addressee.
- **friendships** — readable only if `auth.uid()` is one of the two users. Inserts happen only through the `accept_friend_request` function. Either party can delete (unfriend).

After writing the policies, write a short `supabase/RLS_TESTS.md` listing the specific things I should manually try in order to confirm the policies work — e.g. "open two browser profiles, sign in as two users, confirm user B cannot delete user A's event."

---

## 5. File structure

```
src/
├── main.jsx
├── App.jsx                      routes only
├── index.css                    tailwind + design tokens
├── lib/
│   ├── supabase.js              client init from env vars
│   ├── constants.js             categories, interests, cities
│   └── queries.js               every DB call lives here, nowhere else
├── contexts/
│   └── AuthContext.jsx          session, profile, signUp/signIn/signOut
├── hooks/
│   ├── useEvents.js             fetching + realtime subscription
│   └── useDebounce.js           for the user search input
├── components/
│   ├── Navbar.jsx
│   ├── BottomNav.jsx            mobile only
│   ├── EventCard.jsx
│   ├── InterestTag.jsx
│   ├── ProtectedRoute.jsx
│   ├── EmptyState.jsx
│   └── Skeleton.jsx             loading placeholders
└── pages/
    ├── Landing.jsx              logged-out marketing page
    ├── SignUp.jsx               multi-step onboarding
    ├── SignIn.jsx
    ├── Discover.jsx             main feed
    ├── EventDetail.jsx
    ├── CreateEvent.jsx
    ├── EditEvent.jsx
    ├── Profile.jsx
    ├── EditProfile.jsx
    └── Friends.jsx
```

**Hard rule:** no component calls `supabase` directly. Every query goes through a named, documented function in `lib/queries.js`. This keeps the data layer swappable and makes the code readable.

---

## 6. Features, in build order

### Phase 1 — foundation
Project scaffold, Tailwind config, Supabase client, all SQL migrations, RLS policies, `AuthContext`, `ProtectedRoute`, routing skeleton with placeholder pages. **Stop here.** I'll run the migrations and confirm auth works before you continue.

### Phase 2 — auth flow
- **Landing page** for logged-out users: what the app is, why it exists, sign-up CTA. This is the page I'll send people, so make it good.
- **Sign up**, three steps with a progress indicator: (1) name, email, password — validate password ≥ 8 chars and show a real strength hint, (2) city picker, (3) interests, minimum 3 selected. Pass `display_name` through `options.data` on `signUp` so the trigger picks it up.
- **Sign in** with clear, human error messages. Map Supabase's raw error strings to plain English — "Invalid login credentials" should read "That email and password don't match."
- Handle the email-confirmation state: if Supabase is configured to require confirmation, show a "check your inbox" screen rather than silently failing.

**Stop here.**

### Phase 3 — events
- **Create event** form: title, description, category picker, date, time, end time (optional), city, venue, max attendees (optional), interest tags. Validate that `starts_at` is in the future. Show a live preview of the event card as they type — it makes the form feel less like a chore and encourages better descriptions.
- **Discover feed**: card grid, responsive, single column on mobile. Three sorts — "For you", "Soonest", "Popular". Filter chips for category, and a tag filter row. Filter by city, defaulting to the user's own city with an "all cities" escape hatch.
- **Event detail**: full description, date/time/venue, organizer link, attendee count and avatars, RSVP button, and — the important one — a "3 friends going" line listing which of *your* friends have RSVP'd. That's the network-effect hook; don't skip it.
- **RSVP / un-RSVP**, with optimistic UI updates so the button responds instantly.
- **Edit and delete** for the organizer, with a confirmation dialog on delete.
- Past events must be visually distinct and sorted below upcoming ones.

**Stop here.**

### Phase 4 — social
- **Profile page**: avatar, name, bio, city, interests, events they're hosting, events they're attending (only visible to friends — respect that in the query, not just the UI).
- **Edit profile.**
- **Friends page** with three tabs: friends list, incoming requests, find people.
- **User search**, debounced, case-insensitive partial match on `display_name`.
- Friend request send / accept / decline, using the Postgres function for accept.
- A friend-count badge somewhere in the nav when requests are pending.

**Stop here.**

### Phase 5 — polish
- Realtime subscription on `events` so the feed updates live when someone posts.
- Skeleton loaders on every async view — no blank white screens, ever.
- Empty states with an actual next action, not just "nothing here." An empty feed should say "No events in Chennai yet — post the first one" with a button.
- Error boundary so one broken component doesn't white-screen the whole app.
- 404 page.
- Mobile bottom navigation bar (Discover / Post / Friends / Profile) that replaces the top nav under `md`.
- `og:` meta tags so shared event links preview properly in WhatsApp and Instagram DMs. This directly affects whether the app spreads.

---

## 7. Recommendation logic

Keep it simple and legible, in a single well-commented function. Score each event:

- `+10` per tag overlapping the user's interests
- `+5` if the event city matches the user's city
- `+8` if any friend has RSVP'd (social proof beats interest match — this is deliberate)
- `+min(attendee_count, 15)` as a mild popularity nudge
- `-20` if the event has already started

Sort descending. Write it so I can tune the weights in one place. Add a comment explaining that this is a heuristic, not machine learning, and what a v2 might look like.

---

## 8. Design direction

Do not produce a generic template. Commit to a specific aesthetic and hold it consistently.

**Direction:** editorial print, not SaaS dashboard. Think a well-designed independent magazine — confident typography, generous whitespace, restrained color.

- **Type:** a distinctive display serif for headings (large, tight tracking, italic for emphasis) paired with a clean geometric sans for body. Do not use Inter, Roboto, or system fonts. Do not use Space Grotesk.
- **Color:** warm paper background rather than pure white, near-black ink for text, one confident accent color used sparingly. Avoid purple-on-white gradients and avoid the default Tailwind blue.
- **Layout:** asymmetry is welcome. Big headings. Cards with real breathing room.
- **Motion:** restrained. A staggered fade-in on the feed, subtle hover lift on cards. No bouncing, no spinners where a skeleton would do.
- **Mobile first.** Most users will open this on a phone from an Instagram link. Design for 375px width first, then scale up.

Put design tokens in the Tailwind config as named colors, and build reusable component classes in `index.css` (`.btn-primary`, `.card`, `.chip`, `.input-field`) rather than repeating long class strings.

---

## 9. Also produce

- `.env.example` with `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY`, plus a comment explaining that the anon key is safe to expose publicly *because* RLS is what actually protects the data — and that the service role key must never appear in this project.
- `.gitignore` covering `node_modules`, `dist`, `.env.local`.
- `README.md`: what the app is, the setup steps in order, the schema laid out as a table, and a "how to deploy to Vercel" section.
- `SETUP.md`: numbered, beginner-level walkthrough of creating the Supabase project, running the migrations, enabling email auth, and getting the env vars. Assume the reader has never used Supabase.
- A `seed.sql` with roughly 15 realistic Chennai-based sample events across different categories and dates, so the feed isn't empty while I'm developing.

---

## 10. Cross-platform collaboration

Two people work on this repo: one on macOS, one on Windows. Both run Claude Code. Set the project up so that doesn't cause problems.

- Create a **`.gitattributes`** in the repo root containing `* text=auto eol=lf`. This normalizes line endings so Windows edits don't show up as whole-file changes in git diffs. Do this in Phase 1, before there are many files.
- Commit **`package-lock.json`**. In the README, instruct collaborators to run `npm ci` rather than `npm install` after pulling, so both machines get byte-identical dependency trees. Add a warning not to run `npm audit fix --force`, which silently upgrades major versions and desyncs the two environments.
- Never hardcode absolute paths or platform-specific shell commands in npm scripts. Anything in `package.json` scripts must run identically in zsh and PowerShell.
- Create a **`CLAUDE.md`** in the repo root, since both collaborators' Claude Code sessions read it automatically. It should contain: the commands (`npm ci`, `npm run dev`, `npm run build`), the rule that all database calls live in `lib/queries.js`, the design tokens as literal values, the branch-and-PR workflow, and a note that `.env.local` is never committed. Keep it short — it's loaded into context on every session.

## 11. Working agreement

- After each phase, stop and give me: what you built, what I should test, and anything you're unsure about.
- If a requirement in this spec is ambiguous or you think it's wrong, say so before building it.
- Comment the non-obvious parts — RLS policies, the friendship ordering constraint, the scoring function, and anything involving Supabase-specific behavior.
- Run the build (`npm run build`) at the end of every phase and fix any errors before telling me you're done.
- Do not add analytics, payments, or authentication providers I didn't ask for.
