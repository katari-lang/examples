# concierge — a two-desk community concierge for Discord

A public **face** lives in your community channel and answers questions — but only from notes
its owner has published. The owner curates those notes from a private **control** channel with
three plain commands (`publish`, `retract`, `notes`); when the community asks something the
notes do not cover, the face says it does not know and mails the question to the control
channel, so the owner sees exactly what the community wanted and can publish a note for next
time. What the face knows is exactly what was published — nothing else crosses the membrane.

## What this example teaches

- **Two desks and mail on one region bus** — one request plus one sequential handler per
  serialization domain, and agent-to-agent mail as a fiber whose whole body is one perform:
  [A second agent: desks and mail](https://katari-lang.dev/docs/v0.1/tutorial/a-second-agent).
- **Handler geometry** — why the providers sit above the nursery and the flag bridge above the face
  desk, and why the crash policy's own position is *free* where those two are forced (the file says
  which is which, since the compiler only ever tells you about the forced ones); and failure carried
  back as a *value* (`send_outcome`, `desk_outcome`) instead of a throw across a handler boundary:
  [Handler geometry](https://katari-lang.dev/docs/v0.1/guides/handler-geometry).
- **A resident AI desk with tools** — `ai.advance_desk` advancing one durable conversation per
  arrival, with a per-turn injected notes index and a tool set assembled as data:
  [A Discord bot](https://katari-lang.dev/docs/v0.1/tutorial/a-discord-bot) and
  [Giving the model tools](https://katari-lang.dev/docs/v0.1/tutorial/giving-the-model-tools).
- **A capability membrane in miniature** — the face's tool list *is* the boundary: three reads
  of the shared notes plus a flag, and the control desk is the only writer. Compare the
  full-size version in [tsukasa](https://github.com/yukikurage/discord-bot-example), whose
  public herald is bounded the same way.
- **The memory package as a curated knowledge base** — one shared store workspace, an index the
  model sees every turn, bodies read on demand:
  [Store](https://katari-lang.dev/docs/v0.1/guides/store) and the
  [memory reference](https://katari-lang.dev/packages/memory).
- **Durable execution, and what a restart really costs** — the face's conversation is run state
  (a handler `var`) and **survives a runtime restart**, because nothing here is rebuilt: the
  recovery is one `region.crashed` clause that forks the dead watch again. That is what an FFI
  package looks like when its calls carry the credential instead of a connection handle:
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
channel for the face, and a **private** channel for control — whoever can type in the control
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
you>  notes
bot>  (persistent memory is empty — nothing saved yet.)
you>  publish faq/meetup: The community meetup is every Friday 19:00 at the Kanda office; visitors welcome.
bot>  (published "faq/meetup" — the face answers from it on its next turn.)

── in the public channel ──────────────────────────────────────────────────────
alice> when is the next meetup?
face>  Every Friday at 19:00 at the Kanda office — visitors are welcome!
bob>   is there a hardware channel?
face>  I don't know, sorry — I've flagged your question for the owner.

── back in the control channel ────────────────────────────────────────────────
bot>  (the face was asked something the notes do not cover: "is there a hardware channel?" — `publish <key>: <text>` to teach it.)
you>  publish channels/hardware: Yes — #hardware, ask a mod for access.
```

`retract faq/meetup` makes the face stop knowing it; `help` (or anything unparsed) prints the
command grammar.

## How it is built

Everything is one file, [`src/concierge.ktr`](src/concierge.ktr) — a mini version of the
[second-agent chapter](https://katari-lang.dev/docs/v0.1/tutorial/a-second-agent)'s office, with real
packages on the bus.

- **Two source fibers** (`face_source`, `control_source`) each serve one channel with
  `discord.watch_messages` and report every message as a desk request. The public source tags
  each speaker's id into an HMAC pseudonym (`discord.author_tag`) before it leaves the program,
  and frames the display name as self-chosen — address, never trust.
- **The face desk** is one sequential handler holding one `ai.desk` var. Each arrival runs
  `ai.advance_desk` with the persona riding the provider's `system` parameter and the
  *published-notes index* injected fresh at every model step, so a note published mid-chat is
  visible on the very next turn. Its tool set — the membrane — is
  `[memory.recall, memory.search, memory.list_memories, flag_unknown]`: three reads and a flag,
  no writes.
- **The flag bridge** is the desks' one door to the nursery: the face's `flag_unknown` tool
  performs `flag_question`, and the bridge turns it into mail — a fiber whose whole body is one
  `question_flagged` perform, queued durably behind the current turn.
- **The control desk** is sequential and var-free: it parses each operator line into a command
  *sum* (`publish_note | retract_note | list_notes | show_help`) at the boundary and dispatches
  on the sum. `publish` and `retract` are the program's only writers, through `memory.remember`
  / `memory.forget`; the notes live in the store under one shared workspace
  (`concierge/memory/…`, browsable in the console), so a crashed run can never edit what is
  desired — the notes simply persist.
- **Failure is folded, not thrown**: a model outage becomes a `step_failed` outcome the desk
  reads — the message is filed in the desk's backlog and rejoins the next successful turn, the
  public channel hears one polite line *once per outage*, and the owner gets one line saying
  whether it will heal on its own (`ai.classify_outage`). A dropped public reply comes back as a
  `dropped` value and is reported to the owner; a dropped note *to* the owner is let go, because
  the control channel is where a report of it would have gone. Only non-recoverable failures — a
  bad token, a missing secret — stop the run, loudly.
- **The crash policy** is what makes it a resident on this runtime, and it is one clause. Nothing
  here holds a connection: `discord.provider` serves the bot **token**, resolved per call, and a
  call that needs the gateway (`watch_messages`) opens one for its own lifetime. So a **runtime
  restart** costs exactly the call it interrupted — the watch — which arrives as `region.crashed`,
  and the policy **forks the same watch again**: the fresh call resolves the token and opens its
  own gateway. A crashed one-shot mail is one lost flag and nothing more (the asker was already
  told the face does not know). `region.failed` is the other half: an escaped throw — a revoked
  token, a channel the bot lost access to — is a fault no fresh fork fixes, so it says so in the
  control channel and then **stops the run**. Leaving that source down instead would keep both desks
  standing over a channel nobody is listening to: a concierge that is alive and deaf, which is worse
  than one that stopped, because a stopped run is a fact somebody notices. This is the shape the
  [FFI sidecars guide](https://katari-lang.dev/docs/v0.1/guides/ffi-sidecars#what-a-restart-costs-and-what-a-re-fork-gets-back)
  prescribes: **`crashed` = fork it again, `failed` = stop loudly.** There is no replay scope and
  no panic converter, because there is nothing to rebuild.

Honesty notes:

- **Messages can be missed.** Discord's gateway has no per-event acknowledgement, so across a
  reconnect or a runtime restart a channel message can be missed (and, rarely, repeated) — the
  concierge does not reconcile against channel history. The gap around a restart is the seconds
  between the interrupted watch and its replacement: nothing was listening, so nothing arrived.
- **A restart costs the gap, and nothing else — the conversation survives it.** The face's
  `ai.desk` var is run state, and a recovered run resumes from committed state: since no scope
  here is rebuilt, the chat history and anything filed in its backlog are still there, and the
  next question continues the same conversation. (Under the old `replay.forever` shape this was
  not true — every attempt rebuilt the desk empty. The rule that removed the loop gave the
  conversation back.) The **published notes** were never at risk either way: they live in the
  durable store, which no crash edits. The one thing a restart does take is whichever call was in
  flight, and each crash is one line in the control channel.
- **A crash is forked again with no backoff and no cap**, so a panic that reproduced on *every*
  attempt would be a loop rather than a stop. Nothing an operator can type reaches one here — the two
  channel ids are strings the boot checks, and a gateway that will not open is a `discord_error`
  *throw*, which is `region.failed` and stops the run. Worth knowing anyway, because the control
  channel is the wrong place to watch for such a loop: Discord's rate limit would start dropping those
  very lines (`try_send` folds a drop to nothing), so the channel goes quiet under exactly the failure
  it would have to show. `katari status <run>` and its events are the record that does not thin out.
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
