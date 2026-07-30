# Katari examples

Working, deployable Katari programs — one directory per use case. Each example is a complete
project: clone, add your tokens, `docker compose up`, `katari apply`, and you have a durable
resident agent. Every example pins a published registry snapshot and compiles in CI against the
published CLI, so what you clone is what runs.

If you are new to Katari, start with the [quickstart](https://katari-lang.dev/docs/v0.1/getting-started/quickstart)
and the [tutorial](https://katari-lang.dev/docs/v0.1/tutorial) — the examples here begin where
the tutorial ends, and each README says which guides it puts to work.

| Example | Use case | What it teaches |
| --- | --- | --- |
| [`release-watch`](release-watch/) | A GitHub release monitor you manage from Discord | Durable scheduling, the desired-set (fleet) pattern, store cursors, folding HTTP failures without dying |
| [`standup-scribe`](standup-scribe/) | A Slack standup bot with a human-approved digest | Message plane vs interaction plane, ask controls, approval before posting, scheduled jobs with timezones |
| [`concierge`](concierge/) | A two-desk Discord community concierge | Two desks and mail on one region bus, a curated-knowledge membrane, an AI desk with tools |
| [`inbox-butler`](inbox-butler/) | Gmail triage that proposes calendar events for one-click approval | OAuth credentials and preflight, AI triage with structured output, the approval-gate idiom |

All four are **residents**: long-running programs that stay on a channel or a schedule rather than
running once and exiting. So all four also answer the question a resident cannot avoid — what happens
when the runtime restarts under it. Every package call here carries the credential it acts with, never
a handle into a sidecar's memory, so a restart costs exactly the **one call it interrupted**: the
recovery is a single `region.crashed` clause that forks the dead watcher again, with no supervision
scope and no reconnect to arrange. Each README says plainly what comes through (store state, a desk's
conversation, a half-collected window) and what the interruption costs (the messages nobody was
listening for, an open question). The rule is
[What may cross the boundary](https://katari-lang.dev/docs/v0.1/guides/ffi-sidecars) and the mechanism
under it is in [Durable execution](https://katari-lang.dev/docs/v0.1/concepts/durable-execution).

Reading order: `release-watch` (no model, the durable skeleton alone) → `standup-scribe` or
`concierge` (one model desk each) → `inbox-butler` (multi-provider composition). `concierge` is
where the [tutorial](https://katari-lang.dev/docs/v0.1/tutorial/a-discord-bot) sends you after its
last chapter, so it is the shortest step up from there.

## Running an example

Each example's README has the full walkthrough. The shared shape:

```sh
cd <example>
katari lock                        # fetches the pinned packages into .katari/
(cd .katari/packages/discord-* && npm install)   # the sidecar's own dependencies, once
cp .env.example .env               # then fill in the ids the README lists
docker compose up -d               # runtime + Postgres + blob store, self-hosted
export $(grep '^KATARI_API_KEY=' .env)
katari env set <SECRET> --secret   # per-example tokens; each README lists its own
katari apply
katari run <package>.main --detach
```

The first two lines are the ones a fresh clone needs and a re-run does not. `check`, `build` and
`apply` are offline readers, so nothing fetches the packages until `lock` does; and every example
here depends on `discord` or `slack`, each of which carries an FFI sidecar whose own dependencies
the bundler needs (`slack-*` instead of `discord-*` for `standup-scribe`).

The CLI is the published one: `npm install -g @katari-lang/cli`.

## License

MIT
