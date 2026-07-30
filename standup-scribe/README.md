# standup-scribe

A Slack standup scribe. Every weekday at a configured hour it posts the standup prompt to your
channel; teammates reply in the prompt's thread; when the window closes, one model step drafts a
compact digest and the **facilitator reviews it before it posts** — press *Post it*, rewrite it in
an editable form, or *Discard*. Nothing model-written reaches the channel without a human click,
and what the facilitator submits is exactly what posts. A runtime restart costs the scribe the one
call it interrupted: the day's open window, and the replies already collected in it, both come
through — spelled out under [Durability, honestly](#durability-honestly).

## What this example teaches

- **The two Slack planes** — a message stream for collection, one blocking `ask` for the review —
  and why they cannot race each other:
  [Asking a human](https://katari-lang.dev/docs/v0.1/guides/asking-a-human).
- **An approval gate over an editable draft**: the form's prefill *is* the draft, so "approved"
  means approved at the submitted text, with no re-derivation between the click and the post:
  [Approval gates](https://katari-lang.dev/docs/v0.1/guides/approval-gates).
- **Ask controls as data**, checked where they are built (`check_controls`) and fitted honestly
  (`fit_message`) instead of discovering Slack's caps from a rejected payload.
- **Scheduled jobs with a timezone**: a weekday cron read in an IANA zone, durable timers, and a
  store marker holding the window's identity *and* its lifetime — so the window is a durable fact
  rather than a fiber's private state, and an at-least-once tick cannot open the same day twice:
  [Scheduled jobs](https://katari-lang.dev/docs/v0.1/guides/scheduled-jobs),
  [Store](https://katari-lang.dev/docs/v0.1/guides/store).
- **One-shot model use**: a single `ai.reply` drafts the digest — no chat loop, no tools, one
  failure surface: [Talking to a model](https://katari-lang.dev/docs/v0.1/tutorial/talking-to-a-model).
- **Outcome-as-value folding**: `try_send`'s `delivered | dropped` is read everywhere, a failed
  occurrence becomes one facilitator line, and only auth failures stop the run — loudly:
  [Handler geometry](https://katari-lang.dev/docs/v0.1/guides/handler-geometry).
- **A desk and its region**: a sequential handler as the collection actor, fibers as sources, and a
  crash policy that forks a dead fiber again rather than inventing an answer:
  [A second agent](https://katari-lang.dev/docs/v0.1/tutorial/a-second-agent),
  [A Discord bot](https://katari-lang.dev/docs/v0.1/tutorial/a-discord-bot) (the resident shape).
- **Surviving a runtime restart with a sidecar**: every Slack call carries the tokens it acts with, so
  a restart costs the interrupted call and nothing else — no session to rebuild, no replay scope, and
  the desk's collected replies stay put:
  [FFI sidecars](https://katari-lang.dev/docs/v0.1/guides/ffi-sidecars).

## Setup

### 1. A Slack app (Socket Mode)

Create an app at [api.slack.com/apps](https://api.slack.com/apps) ("From scratch"), then:

1. **Socket Mode**: Settings → Socket Mode → enable it. Generate the app-level token with the
   `connections:write` scope — this is the `xapp-…` token (`SLACK_APP_TOKEN`).
2. **Scopes**: Features → OAuth & Permissions → Bot Token Scopes: add `chat:write`.
3. **Events**: Features → Event Subscriptions → enable, and under "Subscribe to bot events" add
   `message.channels` (plus `message.groups` if your standup channel is private). Socket Mode needs
   no Request URL.
4. **Interactivity**: Features → Interactivity & Shortcuts → toggle it **on**. Without it the
   review's buttons silently do nothing — the two planes are subscribed separately, and this is the
   interaction plane's switch.
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

Store the three secrets in the runtime (each prompts for the value with echo off):

```sh
katari env set SLACK_BOT_TOKEN --secret
katari env set SLACK_APP_TOKEN --secret
katari env set ANTHROPIC_API_KEY --secret
```

Set the non-secret configuration (only the channel is required):

```sh
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
> thread → **jun**: yesterday reviews, today the billing migration, no blockers
>
> **10:00 — #standup-facilitation** (the scribe)
> Standup digest for 2026-07-30 — press **Post it**, open **Edit** to rewrite it, or **Discard**.
> *…the drafted digest…*  [Post it] [Discard] [Edit]
>
> *(the facilitator opens Edit, fixes one line, submits — the submitted text posts to #standup,
> and the review message swaps its controls for an audit line naming who answered)*

If nobody replied in the window, the facilitator instead gets one line — "no entries in the window
— nothing to digest" — and no model call is made.

## How it is built

Everything is in [`src/standup_scribe.ktr`](src/standup_scribe.ktr), top to bottom:

- **Two fibers on one region.** `channel_watcher` serves the standup channel forever and reports
  every message to the desk; `standup_schedule` runs `time.watch` on a weekday cron read in
  `STANDUP_TIMEZONE` and owns the whole occurrence: post the prompt, sleep to an absolute deadline,
  close the window, draft, review, post. One fiber runs all of that deliberately — `time.watch`
  serializes occurrences, and the desk keeps collecting while the fiber sleeps or waits on a human.
- **The window lives in the store, not in the fiber.** `standup/prompts/<date>` holds
  `{ prompt_ts, close_at }` — the thread entries hang under, and the instant collection ends — so one
  reader (`read_marker`) answers `recorded_window | no_window | unreadable_marker` and every path
  dispatches on that. It is what makes three different situations one mechanism: a redelivered tick
  re-opens the same window, a schedule fiber forked again after a crash resumes it before arming its
  watch, and a window whose close has passed is left alone.
- **One sequential desk** (`main`'s `var window` handler) is the collection actor: `idle` or
  `collecting(date, prompt_ts, entries)`. Only replies **threaded under the day's prompt** are
  collected — the `slack.message` data carries its thread's `ts`, so thread membership, not timing,
  is the membership test; top-level chatter is ignored. Author ids are tagged with
  `slack.author_tag` (an HMAC pseudonym under the bot token) before they are stored or shown to the
  model — so the digest names people as `teammate-3f9a2c81` rather than by display name. Slack's
  message event carries only the `U…` id, and the facilitator can put the names back in the Edit box
  before it posts.
- **The review** is one `slack.ask` in the facilitator channel: two buttons and a form whose field
  is **prefilled with the draft**. The answer is read once into `post_approved | digest_discarded`;
  a submitted-but-cleared draft reads as a discard. The ask is raced against a six-hour deadline so
  an unanswered review can never hold back tomorrow's standup.
- **Failure discipline.** Each occurrence runs under one guard: what can heal (a model-step
  failure, a Slack API refusal, a network drop) folds into one facilitator line and the schedule
  stays armed; auth failures and configuration drift re-raise, the fiber's death reaches the
  region's `failed` policy, and the run stops loudly. A `try_send` that drops the approved digest
  reports the approved text to the facilitator so it can be posted by hand.
- **The crash policy is one clause per ending.** `slack.provider` connects nothing — it serves the
  two tokens, resolved per call, and the Socket Mode connection belongs to the one call that needs
  events. So `region.crashed` (a call the runtime interrupted) **forks the same task again**, which
  simply works: the replacement resolves the tokens and opens its own connection. `region.failed` is
  the opposite ending — an uncaught throw is a dead token, a Slack refusal to open the socket, or
  configuration drift — and it stops the run. Nothing wraps any of this in a replay scope, so nothing
  below it is ever rebuilt.

### Durability, honestly

- The cron's next occurrence and the mid-window sleep are durable timers — nothing is held in
  memory, and a deadline that passed while the runtime was down fires at once.
- **A runtime restart keeps the open window *and* the replies already collected in it.** Every Slack
  call carries the tokens it acts with, so a restart invalidates nothing the program holds: it costs
  the one call that was in flight. That is normally the channel watch, which arrives at
  `region.crashed` and is forked again — a fresh Socket Mode connection, collecting again. The desk is
  untouched, because nothing rebuilds it: its `var` is run state, and a recovered run resumes from
  committed state. The schedule fiber is usually untouched too (a mid-window `time.sleep_until` is a
  durable timer, not an external call), and if it *was* in a Slack call, the fork that replaces it
  reads the store marker, re-opens the same window — the desk recognises the same
  `(date, prompt_ts)` and **keeps its entries** — and closes it at the instant the original would
  have. What the restart does cost is the gap: Socket Mode delivers to open sockets, so replies posted
  while nothing was listening are not delivered at all, then or later. The digest is drawn from the
  replies that were collected, and the facilitator can still fix it in the Edit box.
- The tick is **at-least-once**. A redelivered occurrence does *not* repost the prompt, and does not
  re-open a window that has already closed: the per-date marker (`standup/prompts/<date>`, holding
  `{ prompt_ts, close_at }`) is the whole record of today's window. The one gap the marker cannot
  close is a restart that interrupts the prompt's own send — the post may have reached Slack with
  nothing committed to say so. The forked-again fiber finds no marker and stays quiet (that day is
  skipped); a *redelivered tick* inside the window would post a second prompt.
- A marker row this version cannot read — one written by an older build, or hand-edited — is treated
  as "a prompt went out but its lifetime is unknown": nothing is re-posted, that day is skipped, and
  the facilitator is told which row to delete. Upgrading mid-day costs one standup, loudly, rather
  than a duplicate prompt or a crashed fiber.
- Message delivery from Slack is at-least-once too; the scribe keeps no dedup memory, so a reply
  whose acknowledgement was lost to a dropped socket can appear twice in the entries (and the
  digest's model step will see it twice).
- An open review does **not** survive a runtime restart: the ask rides one external call, so a restart
  voids it and its buttons go stale. Nothing invents an answer for a question nobody answered, and that
  day's digest is not posted. The schedule fiber is forked again, finds today's window already closed,
  and says nothing — deliberately: from the store it cannot tell a review that died from a digest that
  posted hours ago, and the honest silence beats a line that guesses. The next weekday asks fresh.
- A standup missed while the runtime was down is **skipped**, not backfilled, and the facilitator gets
  one line saying so. `time.watch` fires a single catch-up for an outage, and the occurrence checks the
  clock before posting: a prompt whose window has already elapsed would only litter the channel with a
  question nobody can still answer.
- Only a failure that will not heal stops the run: an auth error, configuration drift, or a Slack
  refusal to open the socket reaches `region.failed` and throws to the root, which is a stopped run
  somebody notices. A **panic** takes the other road and is forked again, with no backoff — so a
  genuine defect that panics on every attempt is a fork loop rather than a stop. That is why the boot
  validates the hour, the window and the zone: `time.cron` panics on a zone it does not know, and the
  policy would fork the fiber straight back into it. `katari status <run>` and its events are where
  such a loop is visible, not the channel.

## Everyday commands

| Command | What it does |
| --- | --- |
| `katari check` | Compile and report diagnostics |
| `katari apply` | Deploy a new snapshot (compiles what `katari.lock` pins) |
| `katari run [AGENT]` | Start a run and wait (Ctrl-C detaches) |
| `katari ls` | Recent runs (`ls agents`, `ls escalations`, ... for the rest) |
| `katari status <run>` | One run's state, outcome and open questions |
| `katari answer <escalation>` | Answer a question a run escalated |
| `katari cancel <run>` | Cancel a running run |
| `katari env set KEY --secret` | Store a secret programs read via `env.get_secret` |
| `katari add PKG` | Add a dependency from the registry, and re-lock |
| `katari update` | Re-pin to the registry's newest snapshot, and re-lock |
| `katari lock` | Re-lock what `katari.toml` declares (no compile, no deploy) |

`check`, `build` and `apply` all compile the closure `katari.lock` pins, and refuse if you have
edited `katari.toml` since — run `katari lock` and they will tell you what changed.
