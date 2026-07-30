# release-watch

A GitHub release monitor you manage from Discord. Tell it `watch vercel/next.js` in a channel and,
on a durable schedule, it polls the public GitHub releases API (no GitHub account needed) and
announces new releases back in that channel — name, tag, link, and the release notes, honestly
truncated to one Discord message. The watch list lives in the runtime's store, so it survives
restarts and redeploys; a repository that starts failing (a typo'd name, a rate limit) is reported
by `list` without ever taking the monitor down. Deliberately model-free: this example is the
durable-monitor skeleton, with nothing else on the page.

## What this example teaches

- **The fleet custodian pattern** — the desired set (`fleet.register` / `forget` / `desired`) as
  the one durable record of what should be watched, with boot's `sweep` validating or dropping
  every stored row ([Store](https://katari-lang.dev/docs/v0.1/guides/store)).
- **Durable scheduling** — one `time.watch` interval loop whose next occurrence survives restarts
  ([Scheduled jobs](https://katari-lang.dev/docs/v0.1/guides/scheduled-jobs)).
- **Store cursors** — a per-repository cursor cell, written as a sum (`no_baseline` /
  `nothing_released` / `announced_up_to`) so "never checked" and "checked, nothing there" cannot
  blur ([Durable execution](https://katari-lang.dev/docs/v0.1/concepts/durable-execution)).
- **Outcome-as-value folding** — every HTTP failure of a check becomes a value the next `list`
  shows, never a throw that kills the loop
  ([Handler geometry](https://katari-lang.dev/docs/v0.1/guides/handler-geometry)).
- **String boundary → sum dispatch** — channel text is parsed once into a `command` sum; every
  branch after that is a `match`
  ([Language reference](https://katari-lang.dev/docs/v0.1/concepts/language-reference)).
- **Surviving a runtime restart** — the region's crash policy is one clause: a fiber that panicked
  is **forked again**, because every call carries the token it acts with and nothing durable points
  into the sidecar's memory. No replay scope, no panic converter, no session to rebuild
  ([FFI sidecars](https://katari-lang.dev/docs/v0.1/guides/ffi-sidecars)).
- **Desired set vs live state** — an instance's failure never edits the persistent record of what
  is desired: only `unwatch` (a decision) removes a watch
  ([A second agent](https://katari-lang.dev/docs/v0.1/tutorial/a-second-agent),
  [Parallelism](https://katari-lang.dev/docs/v0.1/concepts/parallelism)).

## Setup

### 1. A Discord bot token and a channel

In the [Discord Developer Portal](https://discord.com/developers/applications), create an
application, add a **Bot**, and copy its token. Two settings matter:

- On the Bot page, enable the **MESSAGE CONTENT intent** (the gateway client requests the
  `Guilds`, `GuildMessages` and `MessageContent` intents, and only the last is privileged).
- Invite the bot to your server with permission to read and send messages in the channel you want
  it to serve.

You also need that channel's id: enable Developer Mode in your Discord client's advanced settings,
then right-click the channel and **Copy Channel ID**.

### 2. A local runtime

```sh
cp .env.example .env
echo "KATARI_API_KEY=$(openssl rand -hex 32)"       >> .env
echo "KATARI_SECRET_KEY=$(openssl rand -base64 32)" >> .env
docker compose up -d
export $(grep '^KATARI_API_KEY=' .env)
```

The web console is now at <http://localhost:3000> (it prompts for the `KATARI_API_KEY`).

### 3. Configure and deploy

A fresh clone has no `.katari/`, and `check` / `apply` only read that cache — `katari lock` is what
fills it from what `katari.lock` pins. The `discord` package then wants one `npm install` inside the
fetched copy: it carries an FFI sidecar (a discord.js gateway client) whose own dependencies the
bundler needs.

```sh
katari lock                                    # fetches the pinned closure into .katari/ — check and apply only READ that cache
(cd .katari/packages/discord-* && npm install)

katari env set DISCORD_TOKEN --secret          # paste the bot token at the prompt
katari env set WATCH_CHANNEL <channel id>      # required — commands and announcements live here
katari env set POLL_MINUTES 10                 # optional — poll cadence (default 10; see .env.example)

katari apply
```

### 4. Start the resident

```sh
katari run release_watch.main --detach
```

The bot posts `(release-watch online — watching 0 repositories; ...)` to the channel. A missing
token, an unset `WATCH_CHANNEL`, or a channel the bot cannot post in all fail right there at boot
with one line — never as a bot that looks alive and says nothing. `--detach` prints only the run id,
so that line arrives as the run's result: `katari status <run>` shows it (or start it without
`--detach` the first time and read it in your terminal — Ctrl-C then detaches).

### Try this

In the watch channel:

```text
you:  watch vercel/next.js
bot:  Watching vercel/next.js as `repository-1`. The first check sets the baseline:
      releases published from now on are announced here.

you:  watch typo/nonexistent
bot:  Watching typo/nonexistent as `repository-2`. ...

(up to POLL_MINUTES later, after the first poll)

you:  list
bot:  `repository-1` vercel/next.js — cursor at v15.4.4; last check: baseline set at v15.4.4;
      announcing releases published after it
      `repository-2` typo/nonexistent — no baseline yet; last check: FAILED: GitHub answered 404 —
      no such repository (check the owner/name spelling)

you:  unwatch typo/nonexistent
bot:  Stopped watching typo/nonexistent (`repository-2`).
```

When a watched repository publishes a release, the next poll announces it:

```text
bot:  New release on vercel/next.js: v15.4.5 (v15.4.5)
      https://github.com/vercel/next.js/releases/tag/v15.4.5

      <the release notes, cut to fit one Discord message when long>
```

Restart the runtime (`docker compose restart`) and the monitor comes back by itself, quietly. The
restart interrupts whatever call was in flight — the control fiber's `watch_messages`, always, and a
poll tick if one was mid-request — and each interrupted fiber arrives at `region.crashed`, where the
policy **forks the same task again**: the fresh call resolves the bot token and opens its own gateway
socket, so the channel is served again. Nothing else is rebuilt. Boot does **not** run a second time,
so there is no repeated `(release-watch online — …)` line; the watch list and every cursor are store
rows; a poll fiber that was merely sleeping between ticks is not interrupted at all, since a durable
timer is not an external call. What the restart costs is the commands typed while nothing was
listening — see the delivery note below.

## How it is built

One region bus, two fibers, one desk — all wired in `src/release_watch.ktr`:

- **The control source** (a fiber) serves `WATCH_CHANNEL` with `discord.watch_messages` and
  reports each message to the desk as a `control_message`.
- **The control desk** (a sequential handler in `main`) parses each message at the string boundary
  (`src/release_watch/commands.ktr`) into a `command` sum — `watch` / `unwatch` / `list` /
  anything-else-gets-help — and dispatches on it. Sequential on purpose: `watch` is a
  check-then-register, and the desk's FIFO is the duplicate guard.
- **The poll loop** (the other fiber) is deliberately ONE `time.watch` interval loop over the
  whole desired set, not one fiber per repository — `watch`/`unwatch` therefore need no
  fork/cancel reconcile; the next tick simply sees the new set. Each tick checks every watched
  repository in turn: fetch `releases?per_page=1` via `web.fetch_page`, judge it against the
  cursor, announce with `discord.try_send`, commit cursor + outcome in one store write
  (`src/release_watch/github.ktr`, `src/release_watch/watch_list.ktr`).
- **The store** holds both durable facts, in the app's shared place: `watches/` (the fleet
  registry — what SHOULD be watched; written by the desk, and by boot's sweep only to delete a row
  that no longer reads) and `repositories/<name>` (cursor + last outcome — what HAS been seen;
  written only by the poll loop). On boot, `fleet.sweep` validates every stored row, deletes the
  unreadable ones, and the online line reports the drops.

Failure discipline, honestly stated:

- Every per-repository check failure — 404, rate limit, network fault, unparseable payload — is
  folded into a value stored with the watch and shown by `list`. One repository's failure never
  skips the others and never kills the tick. There are no baked-in retries: the next tick is the
  retry.
- A **new release is announced before the cursor advances**, so announcements are **at-least-once**:
  a crash or a dropped Discord post in that seam means the same release is announced again on the
  next tick, never lost. With `per_page=1`, two releases published inside one poll window are
  announced as one (the newest); the older one is skipped.
- Discord's gateway delivers each message exactly once **within one connection** and promises nothing
  across a reconnect, so a command typed while the bot is reconnecting — or in the seconds between the
  interrupted watch and its replacement fork — can be missed outright. No reply is the symptom; `list`
  is how you check whether it landed.
- A bot reply that Discord drops is not retried or re-reported — the only channel the desk can
  speak in is the one that refused the post; retyping the command is the retry.
- A fiber's **panic** is forked again; a fiber's **throw** stops the run. That split is the whole crash
  policy: a panic means the runtime interrupted an in-flight call, which a fresh call fixes, while a
  throw that escaped both fibers' own folds is a revoked token or a stored shape that stopped parsing —
  one `fatal: ...` as the run's result, because no number of fresh forks fixes a dead credential, and a
  stopped run is a fact somebody notices. `watch_messages` raises that throw itself now: the bot token
  is checked when the gateway opens, not when the provider is installed.
- Unauthenticated GitHub allows 60 requests an hour. Keep `(60 / POLL_MINUTES) * watched
  repositories` under that, or `list` will show rate-limit failures until the window clears.
- The GitHub API rejects any request that carries no `User-Agent`. `web.fetch_page` sets no header of
  its own and relies on the runtime's HTTP client sending one, which the published image does; on a
  runtime build that sends none, GitHub answers 403 and every check shows the rate-limit line.

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
