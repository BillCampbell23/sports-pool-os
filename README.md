# Sports Pool OS

[![tests](https://github.com/OptimusDigitalSuite/sports-pool-os/actions/workflows/test.yml/badge.svg)](https://github.com/OptimusDigitalSuite/sports-pool-os/actions/workflows/test.yml)

**Live app:** https://sports-pool-os.vercel.app

A phone-first NFL pick'em app for running a weekly pool with friends. Most small pools run on group texts and a spreadsheet: someone has to chase picks, score the games, work out who won and track who still owes the buy-in. This app does those jobs. Each player gets a private link with no account or password, makes picks on their phone, and sees live standings. A scheduled job scores the games, sends reminders and keeps the payout ledger.

It runs a real weekly pool for my own group.

## Features

- **Magic-link players.** Each player has a private tokenized URL (`/p/[token]`). There are no accounts or passwords.
- **Weekly picks.** Pick a winner for each game, plus a tiebreaker. Picks lock for the whole week at the first kickoff.
- **Live standings.** Weekly and season standings refresh automatically. Everyone's picks are revealed once the week locks.
- **Commissioner tools.** Add players one at a time or paste a roster. Edit emails, remove players, mark buy-ins paid, set the week lock and game counts, toggle auto-pick, and force a scheduler run.
- **Payout ledger.** Computes the pot and the winner's payout, lists who hasn't paid, and builds Cash App pay links.
- **Scheduled agents.** A Supabase `pg_cron` job calls `/api/agents/tick` every 5 minutes. Four agents run on each tick:
  - *scorekeeper:* syncs scores from the ESPN scoreboard, freezes the week once every game is final, and opens the next week. Standings are computed from picks and games when requested.
  - *autopick:* fills the home team for entered players who missed the deadline.
  - *commissioner:* sends the announcement and two reminders, timed from the week's first kickoff.
  - *recap:* writes the weekly trash-talk recap with Claude. If that call fails, it falls back to a plain summary.
- **Safe to re-run.** Each tick claims a run key first, so overlapping runs do nothing. Pick submission is idempotent.
- **Email** goes through Resend. Without email keys, the app still runs and skips sending.
- **Installable PWA** with a web app manifest and a video backdrop that respects reduced-motion settings.

## Tech stack

| | Version |
|---|---|
| Node.js | **22 or newer** (CI runs Node 22; `@supabase/supabase-js` requires `>=22`) |
| Next.js (App Router, Server Actions) | 16.2.10 |
| React | 19.2.4 |
| TypeScript | 5.9 |
| Tailwind CSS | 4.3 |
| Supabase (Postgres, `pg_cron`, `pg_net`) via `@supabase/supabase-js` | 2.114 |
| Zod (validates ESPN responses) | 4.5 |
| Anthropic SDK (weekly recap) | 0.72.1 |
| Resend (email) | 6.25 |
| Vitest + Testing Library + jsdom | Vitest 3.2.7, RTL 16.3 |
| GitHub Actions | runs `npm ci` and `npm test` on every push to `main` and on every PR |

**Tests:** 473 tests in 48 files. They cover scoring, the payout ledger, the agents, the ESPN parser, server actions and components.

## Getting started

```bash
git clone https://github.com/OptimusDigitalSuite/sports-pool-os.git
cd sports-pool-os
npm ci
```

Create `.env.local` with the following variables. Keep every value secret except the `NEXT_PUBLIC_*` ones.

| Variable | Needed for |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | server-only database access (never expose to the browser) |
| `NEXT_PUBLIC_BASE_URL` | building player links in invites and emails |
| `TICK_SECRET` | authorizes calls to `/api/agents/tick` |
| `RESEND_API_KEY`, `RESEND_FROM` | optional: sending email |
| `ANTHROPIC_API_KEY` | optional: AI recap (falls back to a plain summary) |

Apply the SQL files in `supabase/migrations/` in order. `0002_cron.sql` needs your deployed URL. [`DEPLOY.md`](DEPLOY.md) walks through the full setup.

```bash
npm run dev        # http://localhost:3000
npm test           # vitest run (no database or keys needed)
npm run typecheck  # tsc --noEmit
npm run build
```

## Project layout

```
app/            routes: /p/[token] (picks), /standings, /everyone, /admin, /api/agents/tick
components/     UI components
lib/agents/     scorekeeper, autopick, commissioner, recap + tick runner
lib/nfl/        ESPN scoreboard provider + schema
lib/scoring/    grading and standings
lib/ledger/     pot, payouts, unpaid lines
supabase/       SQL migrations
```

Design notes are in [`DESIGN.md`](DESIGN.md).

---

Built by Bill Campbell with AI coding agents (Claude Code, Codex). I direct the work, then test, fix and ship it.
