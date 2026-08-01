# Katari examples

Working, deployable Katari programs — one directory per use case. Each example is a complete
project: clone, add your tokens, `docker compose up`, `katari apply`, and you have a durable
resident agent. Every example pins a published registry snapshot and compiles in CI against the
published CLI, so what you clone is what runs.

If you are new to Katari, start with the [quickstart](https://katari-lang.dev/docs/v0.1/getting-started/quickstart)
and the [tutorial](https://katari-lang.dev/docs/v0.1/tutorial) — the examples here begin where the
tutorial ends.

| Example | Use case | What it shows |
| --- | --- | --- |
| [`release-watch`](release-watch/) | A GitHub release monitor with a Discord front desk | One AI whose three tools are the watch list, beside a deterministic poll fiber that announces releases at-least-once |
| [`standup-scribe`](standup-scribe/) | A Slack standup bot with a human-approved digest | Message plane vs interaction plane, ask controls, approval before posting, scheduled jobs with timezones |
| [`concierge`](concierge/) | A two-agent Discord community concierge | Two AIs on one route, one `ai.spawn` each, mail between them, a capability membrane made of two tool lists |
| [`inbox-butler`](inbox-butler/) | Gmail triage that proposes calendar events for one-click approval | OAuth credentials, AI triage with structured output, the approval-gate idiom, a region policy that re-forks one fiber kind |

All four are **residents**: long-running programs that stay on a channel or a schedule rather than
running once and exiting. Every package call here carries the credential it acts with, never a handle
into a sidecar's memory, so a runtime restart costs exactly the **one call it interrupted**. Three of
the four answer that call inside the fiber (`use supervise.forever()` plus a panic converter, so the
watch runs again after a backoff and nothing above it is rebuilt); `inbox-butler` answers it at the
region, because its two fiber kinds want opposite answers — a mail watch is re-forked, an unasked
question is not. Each README says what comes through and what the interruption costs. The mechanism is
in [Durable execution](https://katari-lang.dev/docs/v0.1/concepts/durable-execution).

Reading order: `release-watch` (one AI, three tools, one deterministic fiber) → `standup-scribe` (one
model step behind an approval gate) → `concierge` (two AIs on one route, mail between them) →
`inbox-butler` (multi-provider composition). `concierge` is where the tutorial's
[last chapter](https://katari-lang.dev/docs/v0.1/tutorial/a-second-agent) sends you.

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

## Working on the packages alongside an example

The examples pin a published registry snapshot, which is what CI checks and what a clone gets. To
compile one against a package checkout instead, add a `[overrides.<name>] path = "…"` entry per package
in the closure (transitive ones too, and an override may only name a package listed in
`[dependencies].packages`), run `katari lock`, then `katari check` — and strip the overrides and restore
`katari.lock` before committing, so what is pinned here stays what CI compiles.

## License

MIT
