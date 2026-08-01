# concierge — a two-agent community concierge for Discord

Two AIs on one Discord bot. A public **face** lives in your community channel and answers questions —
but only from notes it can read and never write. A private **curator** works with you in a control
channel: you tell it what the community should know, in plain language, and it publishes, rewrites and
retracts the notes. When the community asks something the notes do not cover, the face says it does not
know and **mails the question to the curator**, so you see exactly what the community wanted and can
have a note published for next time. What the face knows is exactly what was published — nothing else
crosses the membrane.

## What this example teaches

- **Several AIs as a region composition** — two `ai.resident` fibers, one per conversation, reaching
  each other by forking `ai.mail` into each other's mailboxes. Adding a third AI here is one line:
  [the ai package's *Several AIs*](https://katari-lang.dev/packages/ai) and
  [A second agent](https://katari-lang.dev/docs/v0.1/tutorial/a-second-agent).
- **The router is the app's, and it is one handler.** Who may address whom is a judgement no package
  can make for you — a fixed pair, a broadcast, a hop limit — so `register` and the app's `send_mail`
  share one `var record[ai.mailbox]` and about ten lines:
  [Handler geometry](https://katari-lang.dev/docs/v0.1/guides/handler-geometry).
- **Handler geometry**, in the one arrangement multi-AI forces: the provider, the ambient notes index
  and the breaker wrap the **root watch**, because a fiber does not inherit the handlers of whoever
  forked it — anything installed beside a fork reaches nobody.
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
packages on the bus.

- **Two `ai.resident` fibers**, forked into one root nursery. `ai.resident` is one AI's whole body —
  its own mailbox nursery, the `register` that announces it, its persona injected at every model step,
  its `serve_observations`, and the mailbox's own watch. Adding an AI is one `region.fork`; the two here
  differ only in their name, their persona, their tool set and where their replies go.
- **Two channel watchers** (`public_watcher`, `control_watcher`), also fibers, each serving one channel
  with `discord.watch_messages`. They do not perform `ai.observation` — a report performed in a watcher
  would reach whatever serving handler is above the *watcher*, which is nobody. They perform the app's
  `send_mail` instead. The public watcher tags each speaker's snowflake into a keyed pseudonym
  (`crypto.pseudonym` under the bot token) before it leaves the program, and frames the display name as
  self-chosen — address, never trust.
- **The router** is one handler holding a `var record[ai.mailbox]`, serving both `ai.register` (an AI
  announcing its inbox) and `send_mail` (anyone addressing one). Delivery is a **fork into the
  addressee's nursery**: `region.fork` routes by the handle alone, so the fiber lands under the
  addressee's `serve_observations` whatever context performed the fork. The task is `ai.mail`, a named
  agent applied to plain data — never an inline closure, because a cross-nursery fork carries its task's
  captured environment into a scope that may die first.
- **The two tool sets are the membrane**, and they are just data. Face:
  `[memory.recall, memory.search, memory.list_memories, flag_unknown]`. Curator: the same three reads
  plus `memory.remember` and `memory.forget`. No write reaches the face, and neither AI holds a channel
  — an agent here speaks only through its `deliver_to`.
- **`flag_unknown`** is the one tool that crosses between them. It reads `ai.inbound_provenance()` —
  the `source` and `hop` of the observation being served — rather than parsing the label the model was
  shown, and mails the question to the curator at one hop more than it arrived with.
- **Three middlewares wrap the root watch**, because a fiber does not inherit the handlers of whoever
  forked it: `anthropic.provider`, then `ai.with_context(inject = published_notes)` — the notes index,
  re-derived fresh at every model step for *both* AIs, because it is the one thing they both work from
  — then `ai.with_breaker`. Reading a step's path from the perform site outward: the breaker decides
  whether to call at all, the index is injected, the provider calls.
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
  one name, a Discord gateway/API failure, the two credential-resolution failures, and a fiber whose
  throw escaped. Every one means the bot cannot speak.

Honesty notes:

- **A conversation is a fiber's own state, and a restart that lands mid-turn takes it.** Everything a
  resident holds — its history, its backlog — lives inside its fiber. A runtime restart between turns
  costs nothing: nothing external is in flight, and the run resumes from committed state. A restart
  *during* a turn interrupts the model call, which panics, which ends the fiber: the crash policy starts
  that AI again (the fresh resident re-registers itself, which is also what repairs the router's handle)
  and the control channel gets one line saying the conversation was lost. **The published notes are
  never at risk either way** — they live in the durable store, which no crash edits, and they are the
  only thing here that is supposed to outlive a conversation.
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
- **Mail is at-most-once, and nothing here folds the loss.** A fork into a nursery whose AI has already
  died panics rather than answering an error. That panic is folded wherever it lands — inside a tool
  dispatch by the turn loop, inside a watcher by that watcher's own supervisor — and the root watch's
  `crashed` clause reports the AI's death one moment later, which is the same news. A program where a
  lost message must be noticed at the *sender* folds it with `supervise.once` + `signal_panics` at the
  router.
- **Nothing damps `hop`.** Mail runs one way here — only the face sends, only the curator receives, and
  the curator has no mail tool — so there is no relay loop to cut and `hop` is provenance the model
  reads. The router is where a `hop >= 2` refusal would go in a program where two AIs can both send.
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
