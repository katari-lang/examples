# standup-scribe

A Slack standup scribe. Every weekday at a configured hour it posts the standup prompt to your
channel; teammates reply in the prompt's thread; when the window closes, one model step drafts a
compact digest and the facilitator reviews it before it posts — press *Post it*, rewrite it in an
editable form, or *Discard*. Nothing model-written reaches the channel without a human click, and
what the facilitator submits is exactly what posts.

## Setup

### 1. A Slack app (Socket Mode)

Create an app at [api.slack.com/apps](https://api.slack.com/apps) ("From scratch"), then:

1. **Socket Mode**: Settings → Socket Mode → enable it. Generate the app-level token with the
   `connections:write` scope — this is the `xapp-…` token (`SLACK_APP_TOKEN`).
2. **Scopes**: Features → OAuth & Permissions → Bot Token Scopes: add `chat:write`.
3. **Events**: Features → Event Subscriptions → enable, and under "Subscribe to bot events" add
   `message.channels` (plus `message.groups` for a private channel). Socket Mode needs no Request URL.
4. **Interactivity**: Features → Interactivity & Shortcuts → toggle it **on**, or the review's
   buttons do nothing: the two planes are subscribed separately.
5. Install the app to your workspace: OAuth & Permissions → Install. The Bot User OAuth Token is
   the `xoxb-…` token (`SLACK_BOT_TOKEN`).
6. `/invite @your-bot` in the standup channel (and the facilitator channel, if different), and copy
   each channel id (the `C…` value in the channel's details).

### 2. Runtime, secrets, config

```sh
cp .env.example .env
echo "KATARI_API_KEY=$(openssl rand -hex 32)"       >> .env
echo "KATARI_SECRET_KEY=$(openssl rand -base64 32)" >> .env
docker compose up -d
export $(grep '^KATARI_API_KEY=' .env)
```

Then the three secrets (each prompts for the value with echo off) and the configuration, of which
only the channel is required:

```sh
katari env set SLACK_BOT_TOKEN --secret
katari env set SLACK_APP_TOKEN --secret
katari env set ANTHROPIC_API_KEY --secret

katari env set STANDUP_CHANNEL          # e.g. C0123456789
katari env set STANDUP_HOUR             # 0-23; default 9
katari env set STANDUP_TIMEZONE         # IANA zone, e.g. Asia/Tokyo; default UTC
katari env set WINDOW_MINUTES           # at least 1; default 60
katari env set FACILITATOR_CHANNEL      # defaults to STANDUP_CHANNEL
```

### 3. Deploy and start the resident

The `slack` package carries an FFI sidecar (a Socket Mode client in TypeScript), so fetch the pinned
packages, give the sidecar its npm dependencies once, and then deploy:

```sh
katari lock                                     # fetches the packages katari.lock pins into .katari/
(cd .katari/packages/slack-* && npm install)    # once: the sidecar's own dependencies
katari apply
katari run standup_scribe.main
```

`Ctrl-C` detaches your terminal; the run keeps serving. Stop it with `katari cancel <run-id>`.

### Try this

With `STANDUP_HOUR=9`, on a weekday:

> **09:00 — #standup** (the scribe)
> Standup time! Reply in this thread within 60 minutes: yesterday / today / blockers.
>
> *(teammates reply in the prompt's thread)*
> thread → **ayaka**: yesterday shipped the importer, today the retry queue, blocked on staging creds
>
> **10:00 — #standup-facilitation** (the scribe)
> Standup digest for 2026-07-30 — press **Post it**, open **Edit** to rewrite it, or **Discard**.
> *…the drafted digest…*  [Post it] [Discard] [Edit]
>
> *(the facilitator opens Edit, fixes one line, submits — the submitted text posts to #standup)*

If nobody replied in the window, the facilitator instead gets one line — "no entries in the window
— nothing to digest" — and no model call is made.

## How it is built

All of it is in [`src/standup_scribe.ktr`](src/standup_scribe.ktr):

- **Two fibers on one region.** `channel_watcher` serves the standup channel and reports every message
  to the desk; `standup_schedule` runs `time.watch` on a weekday cron read in `STANDUP_TIMEZONE` and
  owns the occurrence: post the prompt, sleep to an absolute deadline, close, draft, review, post.
- **The window lives in the store, not in the fiber.** `standup/prompts/<date>` holds
  `{ prompt_ts, close_at }` — the thread entries hang under, and the instant collection ends. One
  reader (`read_marker`) answers `recorded_window | no_window`, and that dispatch covers three cases:
  a redelivered tick re-opens the same window, a fiber run again after a crash resumes it before
  arming its watch, and a window whose close has passed is left alone.
- **One sequential desk** (`main`'s `var window` handler) is the collection actor: `idle` or
  `collecting(date, prompt_ts, entries)`. Only replies threaded under the day's prompt are collected,
  so thread membership rather than timing is the test. Each entry keeps the author's Slack id, and the
  digest writes it as a `<@…>` mention, which Slack renders as that person's name.
- **The review** is one `slack.ask` in the facilitator channel: two buttons and a form whose field is
  prefilled with the draft. The answer is read once into `post_approved | digest_discarded`; a
  submitted-but-cleared draft reads as a discard. The ask is raced against a six-hour deadline, so an
  unanswered review cannot hold back tomorrow's standup.
- **Failure discipline.** Each occurrence runs under one guard: what can heal (a model-step failure,
  a Slack API refusal, a network drop) folds into one facilitator line and the schedule stays armed;
  auth failures and configuration drift re-raise, and the run stops loudly. A `try_send` that drops
  the approved digest reports the approved text so the facilitator can post it by hand.

Three things worth knowing:

- **A runtime restart keeps the open window and the replies already collected in it.** The desk's
  `var` is run state that nothing rebuilds, and the schedule fiber's replacement attempt reads the
  marker, re-opens the same window (the desk recognises the same `(date, prompt_ts)` and keeps its
  entries) and closes it when the original would have. What the gap costs is the replies posted while
  nothing was listening: Socket Mode delivers to open sockets only, so those never arrive.
- **An open review does not survive a restart.** The ask rides one external call, so a restart voids
  it and that day's digest is not posted. The schedule fiber finds today's window already closed and
  says nothing: from the store it cannot tell a review that died from a digest that posted hours ago.
- **A missed standup is skipped, not backfilled.** `time.watch` fires a single catch-up for an
  outage, and the occurrence checks the clock before posting, so a prompt whose window has already
  elapsed is never posted — the facilitator gets one line saying the day was missed.
