# release-watch

A GitHub release monitor with an AI front desk. Ask for `vercel/next.js` in a Discord channel and the
curator — one AI whose whole tool set is `watch`, `unwatch` and `list_watches` — keeps the watch list.
Announcing releases is not its job: a poll fiber checks the watched repositories on a durable timer,
judges each against a stored cursor and posts new releases to the same channel, with no model in that
path. The watch list and every cursor are store rows, so both survive restarts.

## Setup

### 1. A Discord bot and a channel

Create an application in the [Discord Developer Portal](https://discord.com/developers/applications),
add a **Bot**, copy its token, enable the **MESSAGE CONTENT intent**, and invite the bot to a server
with permission to read and send messages in the channel it should serve. For the channel id:
Developer Mode on, right-click the channel, **Copy Channel ID**. The
[Discord bot tutorial](https://katari-lang.dev/docs/v0.1/tutorial/a-discord-bot) walks through it.

### 2. Runtime, config, deploy

```sh
cp .env.example .env
echo "KATARI_API_KEY=$(openssl rand -hex 32)"       >> .env
echo "KATARI_SECRET_KEY=$(openssl rand -base64 32)" >> .env
docker compose up -d
export $(grep '^KATARI_API_KEY=' .env)
```

The web console is now at <http://localhost:3000> (it prompts for the `KATARI_API_KEY`). `katari lock`
fills `.katari/` from what `katari.lock` pins, and the `discord` package wants one `npm install` inside
the fetched copy: it carries an FFI sidecar whose own dependencies the bundler needs.

```sh
katari lock
(cd .katari/packages/discord-* && npm install)

katari env set DISCORD_TOKEN --secret        # the bot token
katari env set ANTHROPIC_API_KEY --secret    # the curator's model key
katari env set WATCH_CHANNEL <channel id>    # required — the curator and the announcements live here
katari env set POLL_MINUTES 10               # optional — poll cadence (default 10)

katari apply
```

### 3. Start the resident

```sh
katari run release_watch.main --detach
```

The bot posts `(release-watch online — …)`. A missing token, an unset `WATCH_CHANNEL` or a channel the
bot cannot post in fails right there at boot with one line, never as a bot that looks alive and says
nothing. `--detach` prints only the run id, so that line arrives as the run's result (`katari status
<run>`).

### Try this

In the watch channel:

```text
you:  keep an eye on vercel/next.js for me
bot:  Watching vercel/next.js. The first check sets the baseline: releases published from now on
      are announced here.

you:  what am I watching?
bot:  vercel/next.js — cursor at v15.4.4; last check: baseline set at v15.4.4; announcing releases
      published after it
```

When a watched repository publishes a release, the next poll announces it. The curator is not involved
and never learns that it happened:

```text
bot:  New release on vercel/next.js: v15.4.5 (v15.4.5)
      https://github.com/vercel/next.js/releases/tag/v15.4.5

      <the release notes, cut to fit one Discord message when long>
```

## How it is built

- `ai.route` opens the nursery everything lives in. `ai.spawn` hires the curator with its three tools,
  its persona, the channel watcher as its `sources` and the channel post as its `deliver_to`.
- The tools are ordinary agents over the store: `watch` and `unwatch` write and delete one key,
  `list_watches` reads them all. The model is the parser — this app has no command grammar.
- The watch list **is** the store: one cell, `watches`, holding a record from `owner/name` to that
  watch's announce cursor and last check outcome. Names are trimmed and lower-cased (GitHub is
  case-insensitive) and nothing else, so `vercel/next.js` keys as itself. Every write is a
  read-modify-write under `store.exclusive`, in the workspace's own lane.
- The poll fiber is forked into the route's nursery with `region.fork` and supervises itself. A tick
  reads the current keys and checks each repository in turn, so `watch` / `unwatch` need no
  fork/cancel reconcile — the next tick simply sees the new set. One check fetches
  `releases?per_page=1`, folds every HTTP failure into a value, judges it against the cursor sum
  (`no_baseline` / `nothing_released` / `announced_up_to`), announces with `discord.try_send` and
  commits cursor and outcome in one write. A failing repository shows in `list_watches`, not in logs.
- The curator's death and the poll fiber's death both stop the run: a conversation lives in its fiber,
  and a restart would have to stand inside the route, where this program deliberately puts none.

Two things worth knowing:

- **Announcements are at-least-once.** The post rides ahead of the state write, and a dropped post
  holds the cursor, so a crash or a refusal in that seam re-announces a release rather than losing it.
  With `per_page=1`, two releases published inside one poll window are announced as one (the newest).
- **Unauthenticated GitHub allows 60 requests an hour.** Keep `(60 / POLL_MINUTES) * watched
  repositories` under that, or every check will show the rate-limit line until the window clears.
