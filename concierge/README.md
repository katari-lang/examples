# concierge — a two-agent community concierge for Discord

Two AIs on one Discord bot. A public **face** lives in your community channel and answers questions —
but only from notes it can read and never write. A private **curator** works with you in a control
channel: you tell it what the community should know, in plain language, and it publishes, rewrites and
retracts the notes. When the community asks something the notes do not cover, the face says it does not
know and **mails the question to the curator**, so you see exactly what the community wanted and can
have a note published for next time. What the face knows is exactly what was published — nothing else
crosses the membrane.

## What this example teaches

- **Several AIs, one line apiece** — `use ai.route()` opens the place they live in, and `ai.spawn` hires
  one per line: its name, its tools, its persona, what it listens to and where it speaks. Adding a third
  AI here is one more `ai.spawn`: [the ai package's *Several AIs*](https://katari-lang.dev/packages/ai)
  and [A second agent](https://katari-lang.dev/docs/v0.1/tutorial/a-second-agent).
- **What the route absorbs, so the program never spells it**: the nursery, its effect ceiling, the
  `record[mailbox]` table, the `register` handshake, the `mail_scope` franking, the delivery fork per
  message and the eviction of a dead AI. This example used to hand-roll every one of those — about a
  hundred lines — and none of them was a fact about *this community*:
  [Handler geometry](https://katari-lang.dev/docs/v0.1/guides/handler-geometry).
- **Handler geometry**, in the one arrangement multi-AI forces: the provider, the ambient notes index
  and the breaker wrap the **route**, because a fiber does not inherit the handlers of whoever forked
  it — anything installed beside a fork reaches nobody.
- **A capability membrane in miniature** — the two tool lists *are* the boundary: the face holds three
  reads of the shared notes plus a flag, the curator holds the same reads plus the two writes. Compare
  the full-size version in [tsukasa](https://github.com/yukikurage/discord-bot-example), whose public
  herald is bounded the same way.
- **The memory package as a curated knowledge base** — one shared store workspace, an index injected at
  every model step for both agents, bodies read on demand:
  [Store](https://katari-lang.dev/docs/v0.1/guides/store) and the
  [memory reference](https://katari-lang.dev/packages/memory).
- **Durable execution, and what a restart really costs** — the notes are durable and no crash edits
  them; a conversation is a fiber's own state and says so, out loud, when it is lost:
  [Durable execution](https://katari-lang.dev/docs/v0.1/concepts/durable-execution) and
  [FFI sidecars](https://katari-lang.dev/docs/v0.1/guides/ffi-sidecars).

## Setup

### 1. A Discord bot and two channels

In the [Discord Developer Portal](https://discord.com/developers/applications), create an
application, add a **Bot**, and copy its token. Two settings matter:

- On the Bot page, enable the **MESSAGE CONTENT intent** — the gateway client requests the
  `Guilds`, `GuildMessages` and `MessageContent` intents, and only the last is privileged.
- Invite the bot to your server with permission to read and send messages in **both** channels
  it will serve.

You also need two channel ids: enable Developer Mode in your Discord client's advanced
settings, then right-click each channel and **Copy Channel ID**. Pick a public community
channel for the face, and a **private** channel for the curator — whoever can type in the control
channel decides what the face knows, so its membership is the whole trust boundary. The bot
refuses to start if the two ids are the same.

### 2. Start a runtime and deploy

```sh
cp .env.example .env
echo "KATARI_API_KEY=$(openssl rand -hex 32)"       >> .env
echo "KATARI_SECRET_KEY=$(openssl rand -base64 32)" >> .env
docker compose up -d
export $(grep '^KATARI_API_KEY=' .env)
```

Fetch the pinned packages and install the Discord sidecar's own dependencies (the `discord`
package carries a TypeScript gateway client; the `npm install`, run once inside the fetched
package, gives the bundler its dependencies). `katari lock` is the step that fetches — `check`
compiles offline and refuses until the locked closure is on disk:

```sh
katari lock
(cd .katari/packages/discord-* && npm install)
katari check
```

Store the credentials and the two channel ids in the runtime's project env — secrets with
`--secret` (the CLI prompts with echo off), plain config without:

```sh
katari env set DISCORD_TOKEN --secret
katari env set ANTHROPIC_API_KEY --secret
katari env set PUBLIC_CHANNEL  <public channel id>
katari env set CONTROL_CHANNEL <private channel id>
```

Deploy and start the resident:

```sh
katari apply
katari run concierge.main
```

`Ctrl-C` detaches your terminal and the run keeps serving; stop it for real with
`katari cancel <run-id>` (find the id with `katari ls`).

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

Ask the curator to `forget faq/meetup` (or just "drop the meetup note") and the face stops knowing
it. The curator speaks plain language, not a command grammar: it has the tools, and what to do with
them is the conversation.

## How it is built

Everything is one file, [`src/concierge.ktr`](src/concierge.ktr), and it is the shape the
[ai package's *Several AIs*](https://katari-lang.dev/packages/ai) section prescribes, with real
packages on the bus. Read the last dozen lines first: they are the whole program.

- **`let root = use ai.route[concierge_effects]()`** opens the place the AIs live in and serves the
  three requests over it — `ai.spawn`, `ai.mail`, `ai.dismiss`. `concierge_effects` is the one type
  argument: the row *this program's* own agents run under, and the route adds its own three requests on
  top of it. Nothing in this file names a nursery, a scope marker, an effect ceiling or a mailbox.
- **Two `ai.spawn` calls, and each parameter is a fact about the community.** `name` is how mail
  addresses it; `tools` is the whole of what it may do; `persona` is who it is, re-derived at every model
  step (an *agent*, so a fixed character is just a constant answer); `deliver_to` is where its words go;
  `sources` is what it listens to. The two AIs differ in exactly those five things.
- **Each channel watcher belongs to an AI**, as one of its `sources` — forked with `self` = that AI's
  name once it is addressable, so a watcher cannot post into a void, and `ai.mail(to = self, …)` is how
  what it hears becomes a turn of that AI's conversation and nobody else's. The public watcher tags each
  speaker's snowflake into a keyed pseudonym (`crypto.pseudonym` under the bot token) before it leaves
  the program, and frames the display name as self-chosen — address, never trust.
- **`ai.mail` is the whole bus.** `flag_unknown` performs it to reach the curator; each watcher performs
  it to reach its own AI. The route does the table lookup and the fork into the addressee's mailbox, so
  no watcher and no tool ever holds another agent's nursery.
- **The two tool sets are the membrane**, and they are just data. Face:
  `[memory.recall, memory.search, memory.list_memories, flag_unknown]`. Curator: the same three reads
  plus `memory.remember` and `memory.forget`. No write reaches the face, and neither AI holds a channel
  — an agent here speaks only through its `deliver_to`.
- **`flag_unknown`** is the one tool that crosses between them. It reads `ai.inbound_provenance()` —
  the `source` and `hop` of the observation being served — rather than parsing the label the model was
  shown, and mails the question to the curator at one hop more than it arrived with.
- **Three middlewares wrap the route**, because a fiber does not inherit the handlers of whoever forked
  it: `anthropic.provider`, then `ai.with_context(inject = published_notes)` — the notes index,
  re-derived fresh at every model step for *both* AIs, because it is the one thing they both work from
  — then `ai.with_breaker`. Reading a step's path from the perform site outward: the breaker decides
  whether to call at all, the index is injected, the provider calls. None of the three serializes: each
  is a parallel handler, so the two AIs' model calls overlap instead of taking turns.
- **The death policy sits above the route**, which is the only place it can be: the route re-raises an
  ending *after* forgetting the AI it belonged to, so a handler below that line never sees one and a
  handler above it can say something true.
- **The store** is one `store.workspace(path = "concierge")` above everything, so the memory facility
  resolves to the same `concierge/memory/…` from the face's reads and the curator's writes alike, and
  its critical sections serialize at that one install's FIFO.

Failure discipline:

- **A model outage costs two lines per channel, not two per message.** `with_breaker` fires
  `on_transition` on the state change and nowhere else — one line to the control channel when the model
  stops answering (carrying the remedy when `outage_kind` says a person can act) and one when it comes
  back. The community's own copy arrives on the other road: `serve_observations` tells the face's
  conversation, which delivers it to the public channel. While the breaker is open, arrivals are held in
  each conversation's bounded backlog and the stored conversation does not grow.
- **A failing tool never fails a conversation.** A panic, a typed throw, malformed arguments or a
  hallucinated name all fold back to the model as a result it reads and corrects from. That is why the
  curator's `memory.forget` needs no guard here for the one throw it has: a note stored in a shape this
  build cannot read comes back to the curator as an error, and the curator tells you.
- **What stops the run is a short list, and it is short on purpose**: a tool set holding two tools of
  one name, a Discord gateway/API failure, the two credential-resolution failures, and either AI
  ending. Every one means the bot cannot speak.

Honesty notes:

- **A conversation is a fiber's own state, and a restart that lands mid-turn takes it — and takes the
  run with it.** Everything a resident holds — its history, its backlog — lives inside its fiber. A
  runtime restart between turns costs nothing: nothing external is in flight, and the run resumes from
  committed state. A restart *during* a turn interrupts the model call, which panics, which ends the
  fiber. The route then forgets that AI and cancels the channel watcher forked beside it, so the
  concierge has gone deaf on that channel; the control channel gets one line saying so and the run
  **stops**, because a bot that looks alive over a channel it cannot hear is the worse ending. Start it
  again with `katari run concierge.main`. **The published notes are never at risk either way** — they
  live in the durable store, which no crash edits, and they are the only thing here that is supposed to
  outlive a conversation.
- **Re-hiring a dead AI is not written here, and the reason is structural.** `ai.spawn` is served *by*
  the route, and the death policy sits *above* it — so a `spawn` performed from that handler does not
  reach the route at all; it escalates to the run root and parks. A program that wants an AI to come back
  by itself puts a second `region.crashed` handler *inside* the route's extent, re-performs the event
  outward (so the route evicts first) and hires it again there. This example prefers the shorter,
  louder ending.
- **Messages can be missed.** Discord's gateway has no per-event acknowledgement, so across a
  reconnect or a runtime restart a channel message can be missed (and, rarely, repeated) — the
  concierge does not reconcile against channel history. The gap around a restart is the seconds
  between the interrupted watch and the attempt that replaces it: nothing was listening, so nothing
  arrived.
- **A watcher supervises itself**, unbounded, with the backoff ceiling as its only brake
  (`supervise.forever()`'s defaults: one second, doubling, capped at fifteen minutes) — so a defect that
  panicked on every attempt would settle into one attempt a quarter-hour rather than stopping. Nothing an
  operator can type reaches one: the two channel ids are strings the boot checks, and a gateway that will
  not open is a `discord_error` *throw*, which is `region.failed` and stops the run. Worth knowing anyway,
  because the control channel is the wrong place to watch for such a loop: Discord's rate limit would
  start dropping those very lines (`try_send` folds a drop to nothing), so the channel goes quiet under
  exactly the failure it would have to show. `katari status <run>` and its events are the record that
  does not thin out.
- **Mail is at-most-once, and the route says so honestly.** Delivery is a fork into the addressee's
  mailbox, and a fork into one that has already died panics; the route folds that panic, answers
  `no_recipient(to)` and forgets the entry on the spot, so the next send fails fast instead of retrying
  into a grave. `flag_unknown` shows the caller's half: it tells the model the flag went nowhere rather
  than claiming it was filed. A *handoff* is all a `posted()` ever means — the report becomes a turn when
  the addressee's serving handler gets to it.
- **Nothing damps `hop`.** Mail runs one way here — only the face sends, only the curator receives, and
  the curator has no mail tool — so there is no relay loop to cut and `hop` is provenance the model
  reads. The route has no opinion about it either: a `hop >= 2` refusal is routing *policy*, which a
  program with one writes over the [mechanism layer](https://katari-lang.dev/packages/ai) the route is
  built from.
- **Nothing escalates to a human.** `katari check` reports the run's only unhandled requests as
  the store's four operations and `io`, which the runtime answers itself — so the concierge never
  parks waiting for an answer.

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
