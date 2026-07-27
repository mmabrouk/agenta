<p align="center">
  <!-- Existing logo block, unchanged -->
</p>

<h1 align="center">The open-source platform for building and running AI agents</h1>

<p align="center">
  Bring your models. Choose your harness. Build agents by working with them — then chat
  with them, share them with your team, or run them in the background.
</p>

<p align="center">
  <a href="https://cloud.agenta.ai?utm_source=github&utm_medium=referral&utm_campaign=readme"><strong>Try Agenta Cloud free</strong></a> ·
  <a href="#self-host-agenta"><strong>Self-host Agenta</strong></a> ·
  <a href="https://agenta.ai/docs/?utm_source=github&utm_medium=referral&utm_campaign=readme"><strong>Documentation</strong></a>
</p>

<!-- DEMO. Same slot as today's YouTube thumbnail: clickable image, directly under the
     header, above everything else. New 30-60s video. One real job, start to finish:
     open an agent → give it a real task → edit its instructions and skills → switch the
     model or harness → put it on a schedule → open the trace and the cost.
     Not an interface tour. -->

<p align="center">
  <!-- Existing badges: MIT, docs, PRs welcome, contributors, PyPI downloads, last commit -->
  <!-- Existing social badges: Slack, LinkedIn, X -->
</p>

---

## Why Agenta?

Most agent tools make you choose. Chat with an agent, but you can't shape it. Code one from
scratch, and you build the whole runtime yourself. Or draw a workflow before you know what
actually works.

Agenta gives you one place to build an agent through real work, improve its instructions and
skills, and then put the same agent to work — for you or for your team.

- **Choose your model** — your API keys, hosted models, OpenRouter, Hugging Face, or your own
  models through Ollama and vLLM.
- **Choose your harness** — run the same agent on Pi or Claude today, with more coming.
- **Choose where it runs** — Agenta Cloud, your own machine, or a Daytona sandbox.
- **Build by working** — chat with the agent and improve it from real results, not from a blank
  config form.
- **Put it to work** — share it with your team, call it from another agent, or run it on a
  schedule or an event.
- **Keep improving it** — every change is versioned, every run is traced, every run has a cost.

## What can you build?

Start from a real job, not an empty agent.

<!-- TODO(mahmoud): replace with templates we can point a stranger at today. Each should link
     to its instructions, skills and tools, a short demo, and a guide to recreating it.
     One of these becomes the launch demo. -->

- **Research agent** — delegates to subagents, gathers evidence, and writes a sourced report.
- **Go-to-market agent** — researches a market, drafts the launch material, improves it with
  your team.
- **Background monitor** — runs on a schedule or an event, investigates, and reports back.
- **Repository agent** — works through issues and code, then hands the repeatable part to a
  background run.

## Not a workflow builder

A workflow follows the graph you drew. It does the same thing every time, in the same order,
and when reality doesn't match the graph, it breaks.

An agent decides what to do next. It tries something, sees the result, and tries something
else. That's the point, and it's why you can't draw it in advance — you find out what works by
doing the work.

If you know every step up front, use a workflow tool. If the job needs judgment, this is for
you.

## What is an agent in Agenta?

An agent is made of parts you can read and change:

```text
Instructions + Skills + Tools and MCP servers = your agent
```

The model, the harness, and the runtime are **choices**, not permanent dependencies. Switch any
of them without redefining what the agent knows or how it works.

Your agent can work in four ways:

- **Interactive** — you work with it directly, in chat.
- **Collaborative** — your team uses the same agent, with the same skills and tools.
- **Composable** — another agent calls it as a subagent.
- **Background** — it runs on demand, on a schedule, or when something happens.

## What's supported today

Checked is what works now. Unchecked is what we're building — [tell us which one you want
next](https://github.com/Agenta-AI/agenta/discussions), or send a PR.

**Harnesses**

- [x] Pi
- [x] Claude
- [ ] Codex
- [ ] OpenCode
- [ ] Your harness — open an issue

**Models**

- [x] Provider API keys (OpenAI, Anthropic, Google, and others)
- [x] OpenRouter
- [x] Self-hosted models — Ollama, vLLM, Hugging Face
- [ ] Your Claude or ChatGPT subscription instead of an API key
- [ ] <verify: anything else>

**Where agents run**

- [x] Agenta Cloud
- [x] Your own machine, self-hosted
- [x] Daytona sandbox
- [ ] <verify: anything else>

**How agents run**

- [x] Interactively, in chat
- [x] On a cron schedule
- [x] On an app event
- [ ] Inbound webhook
- [ ] <verify>

<!-- TODO(mahmoud): every box needs verifying before this ships. An unchecked box is a
     contribution menu. A wrongly-checked box is what kills the launch thread. -->

## From a conversation to a working agent

1. **Start with a real task.** Work with the agent. Watch what it does. Correct it.
2. **Refine how it works.** Change its instructions, skills, tools, model, or harness.
3. **Put it to work.** Keep chatting with it, share it, call it from another agent, or give it a
   schedule.
4. **Learn from every run.** Traces, versions, and cost data, so the next run is better.

## See what your agents actually did

Background agents don't fail loudly. A run reports success, and one step inside it quietly did
nothing. And an agent left alone with an API key can spend a lot of money by morning.

So every run is traced. Every model call, every tool call, every file written, with what it
cost. When something goes wrong at 3am, you open the run and see exactly where — and what it
spent getting there.

- **Tracing** — every agent, model, and tool call.
- **Versioning** — every change to instructions, skills, and configuration. Roll back anything.
- **Cost and usage** — per run, per agent.

<!-- TODO: screenshots. Current README has four; they're all of the old product. Either shoot
     new ones (agent chat, config, trace) or drop the section for launch. -->

## Get started

### Agenta Cloud

The fastest way to try it. Free tier, no credit card.

[**Try Agenta Cloud →**](https://cloud.agenta.ai?utm_source=github&utm_medium=referral&utm_campaign=readme)

### Self-host Agenta

You need Docker and Docker Compose.

1. Clone Agenta:

```bash
git clone https://github.com/Agenta-AI/agenta && cd agenta
```

2. Create the environment file from the example:

```bash
cp hosting/docker-compose/oss/env.oss.gh.example hosting/docker-compose/oss/.env.oss.gh
```

3. Start the services:

```bash
docker compose -f hosting/docker-compose/oss/docker-compose.gh.yml --env-file hosting/docker-compose/oss/.env.oss.gh --profile with-web --profile with-traefik up -d
```

4. Open `http://localhost`.

For remote hosts and custom ports, see the
[self-hosting](https://agenta.ai/docs/self-host/quick-start?utm_source=github&utm_medium=referral&utm_campaign=readme)
and [remote deployment](https://agenta.ai/docs/self-host/guides/deploy-remotely?utm_source=github&utm_medium=referral&utm_campaign=readme)
docs.

## What's rough today

Agenta is early. We would rather you hear it from us than find out at 3am.

<!-- TODO(mahmoud): make every line true. This section earns the self-hosting crowd. Do not
     delete it to make the page look better. -->

- Small local models are unreliable on long tool-calling loops. Frontier models and the larger
  local models hold up. Tell us what you're running.
- Only a handful of MCP servers are wired up out of the box.
- <one more real limitation>

## What's next

- More harnesses and more models
- More MCP servers, tools, and integrations
- Templates you can open with one click
- Better background operation
- Agents that improve themselves from their own traces

Want a specific model, harness, or integration?
[Ask in Discussions](https://github.com/Agenta-AI/agenta/discussions) and we'll add it.

## Community

- [Documentation](https://agenta.ai/docs/?utm_source=github&utm_medium=referral&utm_campaign=readme) — guides and API reference
- [Changelog](https://agenta.ai/docs/changelog?utm_source=github&utm_medium=referral&utm_campaign=readme) — recent updates
- [Slack](https://join.slack.com/t/agenta-hq/shared_invite/zt-37pnbp5s6-mbBrPL863d_oLB61GSNFjw) — ask questions, get help
- [Report a bug](https://github.com/Agenta-AI/agenta/issues)
- [Share an idea](https://github.com/Agenta-AI/agenta/discussions)
- [Contributing guide](CONTRIBUTING.md)

<!-- TODO: seed ~10 issues before launch, several tagged `good first issue`. A stale issue
     tracker repels the exact contributor the checkbox matrix attracts. -->

## Star Agenta

<!-- Existing star CTA block, unchanged -->

## Contributors

<!-- Existing all-contributors table (68 people), unchanged. Real social proof that this is
     not a weekend project. -->

## Disabling Anonymized Tracking

By default, Agenta automatically reports anonymized basic usage statistics. This helps us
understand how Agenta is used and track its overall usage and growth. This data does not
include any sensitive information. To disable anonymized telemetry set
`AGENTA_TELEMETRY_ENABLED` to `false` in your `.env` file.

## License

MIT.
