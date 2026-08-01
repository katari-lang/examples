# concierge — a two-agent community concierge for Discord

Two AIs on one Discord bot. A public **face** lives in your community channel and answers questions —
but only from notes it can read and never write. A private **curator** works with you in a control
channel: you tell it what the community should know, in plain language, and it publishes, rewrites and
retracts the notes. When the community asks something the notes do not cover, the face says so and
mails the question to the curator, so a note can be published for next time.

## Setup

### 1. A Discord bot and two channels

Create the application, add a **Bot** and enable the **MESSAGE CONTENT intent** as in
[A Discord bot](https://katari-lang.dev/docs/v0.1/tutorial/a-discord-bot), then invite it with
permission to read and send messages in **both** channels it will serve. For the ids, enable Developer
Mode in your Discord client and **Copy Channel ID** on each.

Pick a public community channel for the face and a **private** channel for the curator: whoever can
type in the control channel decides what the face knows, so its membership is the whole trust
boundary. The bot refuses to start if the two ids are the same.

### 2. Start a runtime and deploy

```sh
cp .env.example .env
echo "KATARI_API_KEY=$(openssl rand -hex 32)"       >> .env
echo "KATARI_SECRET_KEY=$(openssl rand -base64 32)" >> .env
docker compose up -d
export $(grep '^KATARI_API_KEY=' .env)
```

`katari lock` fetches the pinned packages; the `discord` package carries a TypeScript gateway client
whose own dependencies are installed once inside the fetched copy. Secrets go in with `--secret` (the
CLI prompts with echo off), plain config without:

```sh
katari lock
(cd .katari/packages/discord-* && npm install)
katari env set DISCORD_TOKEN --secret
katari env set ANTHROPIC_API_KEY --secret
katari env set PUBLIC_CHANNEL  <public channel id>
katari env set CONTROL_CHANNEL <private channel id>
katari apply
katari run concierge.main
```

`Ctrl-C` detaches; stop the run for real with `katari cancel <run-id>` (find it with `katari ls`).

### 3. Try this

```text
── in the private control channel ─────────────────────────────────────────────
you>      what do we have on file?
curator>  Nothing yet — the knowledge base is empty.
you>      the community meetup is every Friday 19:00 at the Kanda office, visitors welcome
curator>  Published as faq/meetup. The concierge answers from it on its next turn.

── in the public channel ──────────────────────────────────────────────────────
alice>    when is the next meetup?
face>     Every Friday at 19:00 at the Kanda office — visitors are welcome!
bob>      is there a hardware channel?
face>     I don't know, sorry — I've flagged your question for the team.

── back in the control channel ────────────────────────────────────────────────
curator>  The concierge was asked "is there a hardware channel?" and the notes don't
          cover it. Want me to publish an answer?
you>      yes — #hardware, ask a mod for access
curator>  Published as channels/hardware.
```

Ask the curator to drop the meetup note and the face stops knowing it. It speaks plain language, not a
command grammar: it has the tools, and what to do with them is the conversation.

## How it is built

Everything is one file, [`src/concierge.ktr`](src/concierge.ktr); `serve` is the whole program.

- `let root = use ai.route[concierge_effects]()` opens the place the AIs live in and serves `ai.spawn`,
  `ai.mail` and `ai.dismiss` over it. Nothing in the file names a nursery, a scope marker or a mailbox.
- Two `ai.spawn` calls, and every parameter is a fact about the community: `name` (how mail addresses
  it), `tools`, `persona`, `deliver_to`, `sources`. The two AIs differ in exactly those five things.
- Everything handed to `ai.spawn` is a top-level agent over plain data: the personas, the two
  `deliver_to`s and the two channel watchers, which read their channel id from env. A watcher is forked
  with `self` = its AI's name, so what it hears becomes a turn of that conversation and of no other.
- The public watcher keys each speaker's snowflake into a pseudonym (`crypto.pseudonym` under the bot
  token) and sends it as `author`; the model itself sees only the self-chosen display name.
- The two tool sets are the membrane, and they are just data. Face: `memory.recall`, `memory.search`,
  `memory.list_memories`, `flag_unknown`. Curator: the same three reads plus `memory.remember` and
  `memory.forget`. Neither holds a channel — an AI here speaks only through its `deliver_to`.
- `flag_unknown` is the one tool that crosses between them: it reads `ai.inbound_provenance()` and
  mails the question to the curator at one hop more than it arrived with.
- Three middlewares wrap the route: `anthropic.provider`, `ai.with_context(inject = memory.index_note)`
  — the notes index re-derived at every model step for *both* AIs — and `ai.with_breaker`. None
  serializes, so the two AIs' model calls overlap instead of taking turns.
- The death policy sits above the route, the only place it can say something true: the route re-raises
  an ending *after* forgetting the AI it belonged to. A model outage costs two lines in the control
  channel and not two per message, because `with_breaker` fires on the state change and nowhere else.
- One `store.workspace(path = "concierge")` above everything, so the memory facility resolves to the
  same `concierge/memory/…` from the face's reads and the curator's writes alike.

Honesty notes:

- **A lost conversation stops the run.** A restart during a turn interrupts the model call and ends the
  fiber; the route forgets that AI and cancels the watcher forked beside it, the control channel gets
  one line, and the run stops rather than standing deaf over a channel. Start it again.
- **The published notes survive all of that.** They live in the durable store, which no crash edits,
  and they are the only thing here meant to outlive a conversation.
