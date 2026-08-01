# inbox-butler

A quiet Gmail-to-Calendar secretary. It watches your inbox; when a mail looks like it wants a
meeting — "lunch tomorrow at 12:30?", "the dentist confirms Thursday 09:00" — it drafts the
calendar event and asks you on Discord: title, start, duration and location prefilled in an
editable form, one submit to create it, one click to skip it. It **never writes to your calendar
without your click**, and a mail that is not a meeting costs you nothing at all — no message, no
noise. The watch cursor is durable, so mail that arrives while the butler is down is triaged when
it comes back.

## What this example teaches

- **The approval gate** — fork + ask + one-sum dispatch, Katari's safety story for actions that
  touch the real world: [Approval gates](https://katari-lang.dev/docs/v0.1/guides/approval-gates)
  and [Asking a human](https://katari-lang.dev/docs/v0.1/guides/asking-a-human).
- **Handler geometry** — why the ask adapter sits above the nursery, the spawn handler above the
  desk, and the desk above the watch:
  [Handler geometry](https://katari-lang.dev/docs/v0.1/guides/handler-geometry).
- **AI triage with structured output** — one `ai.infer_structured[T]` call whose object type is the
  contract, converted to a data sum at the boundary:
  [Giving the model tools](https://katari-lang.dev/docs/v0.1/tutorial/giving-the-model-tools).
- **OAuth credentials and a boot preflight** — the runtime owns the Google tokens; the program names a
  credential and a boot check made of `prelude.catch` around `credentials.resolve` fails loud at deploy,
  naming every unset secret at once instead of stopping at the first:
  [Secrets and credentials](https://katari-lang.dev/docs/v0.1/guides/secrets-and-credentials).
- **The converter idiom** — a resident source that folds transient poll failures into a durable
  backoff instead of dying:
  [Scheduled jobs](https://katari-lang.dev/docs/v0.1/guides/scheduled-jobs).
- **Surviving a runtime restart without a supervisor** — one classifier deciding fatal from
  reportable, and a `region.crashed` clause that forks the dead mail watch again, because every call
  carries the credential it acts with and nothing durable points into the sidecar:
  [FFI sidecars](https://katari-lang.dev/docs/v0.1/guides/ffi-sidecars).
- **Exactly-once at the destination** — an at-least-once watch made idempotent with a store marker
  keyed on the message id: [Store](https://katari-lang.dev/docs/v0.1/guides/store) and
  [Durable execution](https://katari-lang.dev/docs/v0.1/concepts/durable-execution).

## Setup

### 1. Start a local runtime

```sh
cp .env.example .env
echo "KATARI_API_KEY=$(openssl rand -hex 32)"       >> .env
echo "KATARI_SECRET_KEY=$(openssl rand -base64 32)" >> .env
docker compose up -d
export $(grep '^KATARI_API_KEY=' .env)
```

The web console is now at <http://localhost:3000>.

### 2. A Discord bot token

In the [Discord Developer Portal](https://discord.com/developers/applications), create an
application, add a **Bot**, and copy its token. Two settings matter:

- On the Bot page, enable the **MESSAGE CONTENT intent** — the gateway client requests it at login.
- Invite the bot to your server with permission to read and send messages in the channel you want
  it to ask in.

You will also need that channel's id: enable Developer Mode in your Discord client's advanced
settings, then right-click the channel and **Copy Channel ID**.

```sh
katari env set DISCORD_TOKEN --secret
```

### 3. A model key

The triage step is one Anthropic call per arriving mail:

```sh
katari env set ANTHROPIC_API_KEY --secret
```

### 4. The Google credential

Both Google packages read **one stored OAuth credential named `google`** — the runtime hosts the
whole flow (browser consent, token exchange, storage, refresh); the program never sees a token.
Open the console at <http://localhost:3000>, go to the project's **Credentials** page, and register
a Google OAuth client named `google` with scopes covering mail reading and event writing, e.g.
`https://www.googleapis.com/auth/gmail.readonly` and
`https://www.googleapis.com/auth/calendar.events`, with `access_type=offline prompt=consent` in the
extra authorize parameters (without them Google issues no refresh token and you re-login hourly).
The full walkthrough, including the Google Cloud side, is in the
[gmail package README](https://katari-lang.dev/packages/gmail) — this example adds nothing to it.

### 5. Configuration

Non-secret values also live in the runtime's env store (not in `.env`, which only compose reads):

```sh
katari env set BUTLER_CHANNEL <channel id>     # REQUIRED — where the butler asks and reports
katari env set BUTLER_TIMEZONE "Asia/Tokyo"    # REQUIRED — the IANA zone "tomorrow 15:00" resolves in
katari env set CALENDAR_ID "primary"           # optional — target calendar (default "primary")
katari env set GMAIL_QUERY "in:inbox"          # optional — the watched search (default "in:inbox")
katari env set GMAIL_POLL_MINUTES 5            # optional — poll cadence (default 5, minimum 1)
```

### 6. Deploy and run

```sh
katari lock                                      # fetches the pinned packages into .katari/ (a fresh
                                                 # clone has none, and check/apply only read the cache)
(cd .katari/packages/discord-* && npm install)   # once: the discord sidecar's own dependencies
katari apply                                     # compiles and bundles the sidecar into the snapshot
katari run inbox_butler.main --detach
```

A missing secret, a bad channel id, an unresolvable timezone or a poll cadence under a minute stops
the boot with one console line naming the fix. If the `google` credential has never been authorized,
the run **parks** on an authorize escalation — open it in the console, log in once, and the run
resumes. A clean boot posts two lines to your channel: `(inbox-butler online — clock Asia/Tokyo)`
from the boot check, then `(watching your inbox)` from the resident itself. Both are posted once per
run, and a runtime restart does not repeat them — nothing about the resident is re-run.

### Try this

Send yourself a mail:

> **Subject:** Lunch Thursday?
> **Body:** Want to grab lunch this Thursday at 12:30 at Blue Bottle Shibuya? Should take an hour.

Within one poll interval the butler posts to `BUTLER_CHANNEL`:

> Mail from alice@example.com — "Lunch Thursday?" looks like it wants a meeting: Alice proposes
> lunch on Thursday and names a time and place.
> Proposed: Lunch with Alice — 2026-08-06T12:30:00+09:00, 60 min at Blue Bottle Shibuya.
> Open "edit & create" to fix any field before it lands, or skip.
> `[edit & create]` `[skip]`

Open **edit & create**, change the duration to `90`, submit — the butler replies
`(created: Lunch with Alice — … — https://www.google.com/calendar/event?eid=…)` and the event is on
your calendar **exactly as you submitted it**, not as the model proposed it. Press **skip** instead
and nothing happens anywhere — no event, no message. A newsletter arriving in the same window
produces nothing at all: the butler is quiet by design.

## How it is built

One region, one desk, two kinds of fiber — all in
[`src/inbox_butler.ktr`](src/inbox_butler.ktr):

- **The source fiber** (`mail_source`) runs `gmail.watch` with a durable cursor
  (`inbox_butler/gmail_cursor`), reporting each newly arrived mail to the desk as one
  `mail_arrived` request. It is wrapped in `supervise.forever` plus a converter handler, so a Gmail
  outage or an expiring token becomes a capped backoff instead of a dead watch.
- **The desk** (a sequential `mail_arrived` handler) triages one mail at a time: a durable
  `inbox_butler/handled/<id>` marker first (the watch is at-least-once; the marker holds the model
  call and the question to at most one per mail — the fine print below says what that costs), then
  `triage.triage_mail` —
  [`src/inbox_butler/triage.ktr`](src/inbox_butler/triage.ktr), one `ai.infer_structured` call
  returning `not_event | event_candidate | triage_failed`. The desk dispatches on that sum and
  nothing else: quiet, report, or gate.
- **The gate** (`event_gate`, forked through `spawn_gate`) owns one mail's whole
  question-and-consequence: a prefilled editable form plus a skip button, and on submit
  `google_calendar.create_event` with exactly the submitted values. The desk never waits — the
  human wait holds only its own fiber, and the ask adapter is a `parallel` handler so several
  gates can wait at once.
- **Failure is data end to end.** `ask_operator` answers `asked | not_asked` (a question Discord
  refused is a value the gate reads, not a throw it could never catch); a failed `create_event`
  becomes one typed line in the channel; a fiber's death arrives at `region.failed`, and one
  `classify` — the single place the unhealable set is named — decides fatal (a revoked credential:
  the run stops loudly) versus one reported errand. The calendar package carries no location field,
  so the location box rides the event description.
- **No supervisor, and nothing to supervise.** Every package call carries the credential it acts with,
  so nothing the resident installs can be invalidated by a runtime restart — there is no connection to
  rebuild and therefore no attempt loop. Each interrupted call is answered where it happens instead:
  the **mail watch** arrives at `region.crashed` and is **forked again** (its cursor is a store row, so
  the downtime is delivered, not skipped); a pending **ask** becomes `not_asked` at the adapter, which
  the gate reports and the desk survives; an interrupted **http** call is a typed `fetch_error` the
  triage guard already folds. A gate's panic is reported rather than re-forked, because re-asking a
  question the operator may already have answered is worse than saying nothing was asked. See
  [FFI sidecars](https://katari-lang.dev/docs/v0.1/guides/ffi-sidecars#what-a-restart-costs-and-what-a-re-fork-gets-back).

Honest fine print:

- `gmail.watch` reports **arrivals only** — the backlog present before the first run is never triaged,
  and delivery is at-least-once.
- The marker is written **before** the triage call, so a mail is asked about **at most once**: a
  redelivery costs neither a second model call nor a second question. The price runs the other way — a
  mail the butler could not triage (a Gmail or model blip) is reported to the channel once and **not
  tried again**, and a mail whose triage the runtime interrupted is marked handled and skipped. Both
  stay in your inbox, which is where you would have read them anyway.
- A pending question does **not** survive a runtime restart: the interrupted ask comes back as "the
  question never reached the operator", the gate says so in the channel, and the desk carries on. The
  posted controls go stale and that candidate is not re-asked — the mail is still in your inbox, so
  nothing is lost silently.
- **A lost errand is one line, not a restart.** A Google 5xx that escaped a gate, a network drop, an
  unforeseen failure — each is reported in the channel (`(gate:mail:… failed: …. That errand did not
  complete; the butler keeps running.)`) and the next mail is unaffected. Nothing is re-tried behind
  your back, because a retry of a half-finished errand is how a calendar gets two of the same event.
- **What a runtime restart keeps, and what it costs.** It keeps everything: the store's
  `inbox_butler/handled/<id>` markers and the gmail cursor, so the mail that arrived meanwhile is
  triaged oldest first and what was handled stays handled. What it costs is the one call it interrupted
  — normally the mail watch, which is forked again with one line in the channel
  (`(the mail watch stopped: …. Starting it again …)`), and any question standing in the channel, which
  goes stale and reads `(expired)`; its mail is still in your inbox. There is no attempt to restart and
  no reconnect to wait for: `(watching your inbox)` is posted once per run, not once per recovery.
- Only what nothing can heal ends the run: a revoked Discord token, a Google 401/403 while triaging
  or creating, a secret that has been unset. That is one `fatal:` console line, deliberately — a butler
  that looks alive and quietly does nothing is worse. (A Google credential whose refresh has died is
  not this case: it parks on a re-authorization prompt, as in step 4.) A **panic** on the resident's own
  path — a Discord post the runtime interrupted between the desk and the channel — ends the run the same
  visible way, as one `failed:` line, rather than being retried in silence.
- **A source panic is forked again with no backoff**, so a defect that panics on *every* attempt is a
  fork loop rather than a stop. That is why the boot validates the timezone and the poll cadence: a
  cadence under a minute panics `time.interval` on every fork, and the loop would report a line to the
  channel each time — which Discord's rate limit then starts dropping, so the channel is the first
  thing to go quiet under exactly the failure it is meant to show. `katari status <run>` and its events
  are the record that does not thin out.
- Discord replies use `try_send`, and a confirmation that cannot be posted is deliberately dropped: this
  channel is the reporting surface, so a line about the channel has no second audience.

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
