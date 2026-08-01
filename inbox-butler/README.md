# inbox-butler

A quiet Gmail-to-Calendar secretary. It watches your inbox; when a mail looks like it wants a
meeting — "lunch tomorrow at 12:30?", "the dentist confirms Thursday 09:00" — it drafts the
calendar event and asks you on Discord: title, start, duration and location prefilled in an
editable form, one submit to create it, one click to skip it. It never writes to your calendar
without your click, and a mail that is not a meeting costs nothing at all — no message, no noise.
The watch cursor is durable, so mail that arrives while the butler is down is triaged when it returns.

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

Create the application, add a **Bot** and enable the **MESSAGE CONTENT intent** as in
[A Discord bot](https://katari-lang.dev/docs/v0.1/tutorial/a-discord-bot), then invite it with
permission to read and send messages in the channel you want it to ask in. For that channel's id,
enable Developer Mode in your Discord client and **Copy Channel ID**.

```sh
katari env set DISCORD_TOKEN --secret
katari env set ANTHROPIC_API_KEY --secret
```

### 3. The Google credential

Both Google packages read **one stored OAuth credential named `google`** — the runtime hosts the whole
flow and the program never sees a token. On the console's **Credentials** page, register a Google OAuth
client named `google` with `https://www.googleapis.com/auth/gmail.readonly` and
`https://www.googleapis.com/auth/calendar.events`, and `access_type=offline prompt=consent` in the extra
authorize parameters (without them Google issues no refresh token). The full walkthrough is in the
[gmail package README](https://katari-lang.dev/packages/gmail) — this example adds nothing to it.

### 4. Configuration

Non-secret values also live in the runtime's env store (not in `.env`, which only compose reads):

```sh
katari env set BUTLER_CHANNEL <channel id>     # REQUIRED — where the butler asks and reports
katari env set BUTLER_TIMEZONE "Asia/Tokyo"    # REQUIRED — the IANA zone "tomorrow 15:00" resolves in
katari env set CALENDAR_ID "primary"           # optional — target calendar (default "primary")
katari env set GMAIL_QUERY "in:inbox"          # optional — the watched search (default "in:inbox")
katari env set GMAIL_POLL_MINUTES 5            # optional — poll cadence (default 5, minimum 1)
```

### 5. Deploy and run

```sh
katari lock                                      # fetches the pinned packages into .katari/
(cd .katari/packages/discord-* && npm install)   # once: the discord sidecar's own dependencies
katari apply                                     # compiles and bundles the sidecar into the snapshot
katari run inbox_butler.main --detach
```

A bad channel id, an unresolvable timezone or a poll cadence under a minute stops the boot with one
console line naming the fix; a clean boot posts `(inbox-butler online — clock Asia/Tokyo)` once per run.
The first poll resolves the `google` credential: never authorized, and the run **parks** on an
authorize escalation — open it in the console, log in once, and the run resumes.

### Try this

Send yourself a mail:

> **Subject:** Lunch Thursday?
> **Body:** Want to grab lunch this Thursday at 12:30 at Blue Bottle Shibuya? Should take an hour.

Within one poll interval the butler posts to `BUTLER_CHANNEL`:

> Mail from alice@example.com — "Lunch Thursday?" looks like it wants a meeting: Alice proposes
> lunch on Thursday and names a time and place.
> Proposed: Lunch with Alice — 2026-08-06T12:30:00+09:00, 60 min at Blue Bottle Shibuya.
> `[edit & create]` `[skip]`

Open **edit & create**, change the duration to `90`, submit — the event lands on your calendar exactly
as you submitted it, not as the model proposed it. Press **skip** and nothing happens anywhere.

## How it is built

One region, one desk, two kinds of fiber, all in [`src/inbox_butler.ktr`](src/inbox_butler.ktr):

- **The source fiber** (`mail_source`) runs `gmail.watch` with a durable cursor
  (`inbox_butler/gmail_cursor`), reporting each newly arrived mail as one `mail_arrived` request.
  `supervise.forever` plus a converter turns a Gmail outage or an expiring token into a capped backoff.
- **The desk** (a sequential `mail_arrived` handler) triages one mail at a time: a durable
  `inbox_butler/handled/<id>` marker first, then `triage.triage_mail` in
  [`triage.ktr`](src/inbox_butler/triage.ktr) — one `ai.infer_structured` call returning
  `not_event | event_candidate | triage_failed`. The desk dispatches on that sum and nothing else.
- **The gate** (`event_gate`, forked through `spawn_gate`) owns one mail's whole
  question-and-consequence: a prefilled editable form plus a skip button, and on submit
  `google_calendar.create_event` with exactly the submitted values. The desk never waits, and the
  ask adapter is a `parallel` handler so several gates can wait at once.
- **Blank means blank.** Every box ships the model's proposal as prefill, so a box that comes back
  empty was cleared on purpose: an emptied title is a refusal, an emptied start creates nothing.
- **Failure is data end to end.** `ask_operator` answers `asked | not_asked` — a question Discord
  refused is a value the gate reads, not a throw it could never catch. A failed `create_event`
  becomes one typed line in the channel, and a fiber's death arrives at `region.failed`, where a
  rejected credential stops the run and everything else is one reported errand.
- **No supervisor, and nothing to supervise.** Every package call carries the credential it acts with,
  so nothing the resident installs can be invalidated by a restart. Each interrupted call is answered
  where it happens: the mail watch arrives at `region.crashed` and is forked again, a pending ask
  becomes `not_asked` at the adapter, an interrupted http call is a `fetch_error` the triage guard folds.

Honest fine print:

- The marker is written **before** the triage call, so a mail is asked about **at most once**: a redelivery
  costs neither a second model call nor a second question. The price runs the other way: a mail the
  butler could not triage, or whose triage was interrupted, is never tried again — it stays in your inbox.
- A pending question does **not** survive a runtime restart: the interrupted ask comes back as "the
  question never reached the operator", the gate says so in the channel, and the desk carries on.
  The posted controls go stale and that candidate is not re-asked.
