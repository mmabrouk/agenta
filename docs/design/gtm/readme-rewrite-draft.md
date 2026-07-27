<p align="center">
  <a href="https://agenta.ai?utm_source=github&utm_medium=referral&utm_campaign=readme">
      <picture>
        <source width="275" media="(prefers-color-scheme: dark)" srcset="https://github.com/user-attachments/assets/97e31bfc-b1fc-4d19-b443-5aedf6029017">
        <source width="275" media="(prefers-color-scheme: light)" srcset="https://github.com/user-attachments/assets/fdc5f23f-2095-4cfc-9511-14c6851c1262">
        <img alt="Agenta logo" src="https://github.com/user-attachments/assets/fdc5f23f-2095-4cfc-9511-14c6851c1262">
      </picture>
  </a>
</p>

<h1 align="center">The open-source workspace for building agents and AI automations</h1>

<p align="center">
  Build agents by chatting with them. Share them with your team or run them in the background.
</p>

<p align="center">
  <a href="https://cloud.agenta.ai?utm_source=github&utm_medium=referral&utm_campaign=readme"><strong>Try Agenta Cloud free</strong></a> &bull;
  <a href="#self-host-agenta"><strong>Self-host Agenta</strong></a> &bull;
  <a href="https://agenta.ai/docs/?utm_source=github&utm_medium=referral&utm_campaign=readme"><strong>Documentation</strong></a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT license." />
  <a href="https://agenta.ai/docs/?utm_source=github&utm_medium=referral&utm_campaign=readme">
    <img src="https://img.shields.io/badge/Doc-online-green" alt="Doc">
  </a>
  <a href="https://github.com/Agenta-AI/agenta/blob/main/CONTRIBUTING.md">
    <img src="https://img.shields.io/badge/PRs-Welcome-brightgreen" alt="PRs welcome" />
  </a>
  <img src="https://img.shields.io/github/contributors/Agenta-AI/agenta" alt="Contributors">
  <a href="https://pypi.org/project/agenta/">
    <img src="https://img.shields.io/pypi/dm/agenta" alt="PyPI - Downloads">
  </a>
  <img src="https://img.shields.io/github/last-commit/Agenta-AI/agenta" alt="Last Commit">
</p>

<p align="center">
    <a href="https://join.slack.com/t/agenta-hq/shared_invite/zt-37pnbp5s6-mbBrPL863d_oLB61GSNFjw">
        <img src="https://img.shields.io/badge/JOIN US ON SLACK-4A154B?style=for-the-badge&logo=slack&logoColor=white" />
    </a>
    <a href="https://www.linkedin.com/company/agenta-ai/">
        <img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white" />
    </a>
    <a href="https://twitter.com/agenta_ai">
        <img src="https://img.shields.io/twitter/follow/agenta_ai?style=social" height="28" />
    </a>
</p>

---

## What is Agenta?

Agenta gives your team one place to build, run, and improve agents and AI automations.

You create an agent by chatting with it and explaining the work. You test it on real tasks, correct it, and improve its instructions, skills, tools, and context over time. Once it works, your team can keep using it in chat or run it in the background on a schedule or event.

The people who understand the work should be able to build the agents that do it. They should not need to wait for another internal tool or translate their work into a rigid workflow first.

## Why we built Agenta

We built Agenta for ourselves.

We were already using Claude Code and other coding agents for work outside engineering. We used them for go-to-market research, positioning, messaging, and recurring operations. The agents were powerful, but the setup lived in an integrated development environment or terminal. It was difficult to share with the team, keep context over time, run in the background, and understand what happened after a run.

We wanted:

- One workspace where each agent could keep files and build durable context, such as a go-to-market wiki.
- Agents that the whole team could use, not only the person who created them.
- The freedom to choose and switch models and agent harnesses. A harness is the coding agent runtime, such as Claude Code or Pi, that plans and uses tools.
- Support for open formats and standards such as `AGENTS.md`, skills, tools, and Model Context Protocol (MCP) servers.
- Versions, traces, token usage, and estimated costs so we could understand runs and improve agents over time.
- Interactive agents that could become background automations without rebuilding the task as a node graph.

So we built Agenta. We have been using it while building it, first for go-to-market and engineering work. It is still early, and we want to build the next set of agents and templates with the community.

## What sets Agenta apart?

### Build agents by working with them

You do not need to define the entire automation before you start. Give the agent a real task, see what it does, and improve it through conversation.

When the agent works, keep using it interactively, share it with teammates, or add a trigger so it runs in the background.

### Choose your harness and model

Agenta separates the agent from the model and harness that run it. This gives you control and reduces dependence on one vendor.

Today, the launch build supports Pi and Claude Code harnesses. It supports selected hosted model providers, including OpenRouter. More harnesses and model paths are planned.

### Use an existing subscription when self-hosting

For supported local self-hosted configurations, you can use an existing Claude or ChatGPT/Codex login instead of paying for every run through a separate API key.

This requires mounting the local harness configuration into the runner and logging in on the host. Subscription state stays on the local runner and does not run on Daytona.

### Keep agents portable and inspectable

An agent in Agenta is made from components you can read and change:

```text
AGENTS.md + Skills + Tools and MCP servers + Triggers + Workspace files
```

These components describe what the agent knows, what it can do, when it runs, and which context it keeps. They are not hidden inside a proprietary visual workflow.

Open formats make the agent easier to inspect and reuse. Exact portability still depends on what each harness supports. For example, MCP support can vary by harness and transport.

### Give every agent a workspace

An agent can access folders and files in its workspace. This gives it durable working context instead of making every conversation start from zero.

For example, our go-to-market agent works with a wiki that contains competitor research, positioning, messaging, experiments, and founder decisions. It can update that context as the work changes and use it in later tasks.

A workspace can contain:

- Research and source material
- Company and product context
- Instructions and reusable skills
- Drafts and completed work
- A living wiki maintained with the agent

### Version agents and inspect every run

Agents are non-deterministic. The same instruction can produce different actions and results. Agenta records the information you need to understand those runs and improve the agent.

- **Tracing:** inspect agent, model, and tool calls.
- **Versioning:** keep an immutable history of instructions, skills, tools, and configuration.
- **Usage:** inspect tokens for each run.
- **Estimated cost:** see list-price estimates when the model is recognized.
- **Evaluations:** compare behavior as you change an agent.

These controls help you build agents that become more reliable over time. They do not guarantee that every agent run will succeed.

### Share agents with your team

You can share agents so teammates use the same instructions, skills, tools, and workspace context. This turns a setup that lived in one person's terminal into a shared system for the team.

Role-based access control (RBAC), single sign-on, and related organization controls are enterprise features.

## What is an agent in Agenta?

Your agent combines six parts:

1. **Instructions:** an `AGENTS.md` file that describes its role and how it should work.
2. **Skills:** reusable knowledge and procedures the agent can load when needed.
3. **Tools:** actions the agent can take in Agenta or external services.
4. **MCP servers:** external tools exposed through the Model Context Protocol.
5. **Triggers:** schedules or connected application events that start the agent.
6. **Workspace files:** persistent context and work products the agent can read and update.

The model, harness, and runtime remain separate choices.

Your agent can work in four ways:

- **Interactive:** you work with it directly in chat.
- **Collaborative:** teammates use the same agent and shared context.
- **Composable:** another agent calls it as a subagent.
- **Background:** it runs on demand, on a schedule, or when a connected event occurs.

## From conversation to automation

1. **Start with a real task.** Explain the work and let the agent attempt it.
2. **Correct the result.** Add context and clarify how decisions should be made.
3. **Improve the agent.** Update instructions, skills, tools, files, model, or harness.
4. **Save the working version.** Keep the configuration that produced the useful result.
5. **Share or automate it.** Let teammates use it, or run it on a schedule or event.
6. **Inspect the runs.** Use traces, versions, token usage, costs, and evaluations to improve it.

## What can you build with Agenta?

Agenta is new, so this library is still thin. We would like contributions, templates, and examples from people using it on real work. We will keep this section updated as those examples prove useful.

### Go-to-market agent

Give the agent access to a workspace containing competitors, positioning, messaging, customer research, experiments, and decisions. Use it to research markets, draft launch assets, maintain the wiki, and prepare recurring updates.

This is one of the agents we use ourselves.

### Signup research and outreach agent

When someone signs up, start an agent through a connected application event. The agent researches the company, compares it with your customer profile, and prepares an outreach recommendation.

### Engineering agent

Give the agent repository context, tools, and engineering procedures. Use it to investigate issues, prepare implementation work, review changes, or run recurring repository checks.

We have also been automating parts of our engineering work with Agenta.

### Research agent

Ask an agent to gather evidence, delegate parts of the research, preserve sources, and write a report into its workspace. Later runs can build on the same research instead of starting again.

### Support agent

Connect the agent to support context and product documentation. It can investigate a question, prepare a response, and preserve useful findings for the team.

## Templates and tutorials

We plan to ship each launch tutorial with the agent template it builds. The tutorial proves that the template works, and the template gives the reader a working starting point.

Initial candidates:

- Build a go-to-market research agent with a persistent wiki
- Research new signups and prepare outreach recommendations
- Turn a recurring engineering check into a scheduled agent
- Build a support investigation agent with shared team context

## Not another workflow builder

A workflow runs the graph you define. That works well when every step is known in advance and should run in the same order.

An agent can decide what to do next. It can use tools, write code when needed, inspect the result, and change its approach. You discover the useful process by working with the agent instead of drawing every branch before the first run.

If you know every step in advance and need deterministic execution, use a workflow tool. If the work needs judgment, research, iteration, or changing context, an agent may fit better.

Agenta can still show you what the agent did through traces and visualizations. The visualization explains a run. It does not force you to design the work as a node graph first.

## How does Agenta compare?

### n8n, Activepieces, Gumloop, and Zapier

These products center on workflows and application integrations. You usually define steps, nodes, or recipes that connect services.

Agenta centers on agents that you build through conversation and real work. The agent can plan, use tools, write code, maintain workspace context, and adapt its approach. Agenta also versions the agent and traces each run.

Use a workflow product when the process is known and deterministic. Consider Agenta when the task needs judgment and you want the same agent to work interactively and in the background.

### Claude Cowork

Claude Cowork provides a workspace experience built around Claude. Agenta's intended difference is control and portability: open-source deployment, multiple supported harnesses and models, inspectable agent components, team sharing, triggers, versioning, and tracing.

### Claude Code, Codex, Pi, and OpenCode

These are coding-agent tools or harnesses, not direct replacements for Agenta. They provide the execution layer that plans and uses tools.

Agenta adds the shared workspace around supported harnesses: agent configuration, files, team access, triggers, versions, traces, evaluations, and reusable templates. The goal is to turn the power people already use individually into agents the whole team can use.

## Tools, MCP servers, and integrations

Agents can use harness-native tools, Agenta platform tools, configured MCP servers, and connected application integrations.

Agenta can connect to services such as GitHub and Slack through its integration provider. Connected application events can also trigger an agent.

## What's supported today

**Harnesses**

- [x] Pi
- [x] Claude Code
- [ ] Codex as a harness
- [ ] OpenCode
- [ ] More harnesses

**Models and authentication**

- [x] Selected provider API keys, including OpenAI, Anthropic, Google, Mistral, Groq, MiniMax, Together AI, and OpenRouter through supported harnesses
- [x] Claude and ChatGPT/Codex subscription authentication for self-hosted local runs with manual configuration
- [ ] Ollama for agent harness runs
- [ ] vLLM for agent harness runs
- [ ] Hugging Face models for agent harness runs
- [ ] Any OpenAI-compatible endpoint

**Where agents run**

- [x] Agenta Cloud
- [x] Self-hosted through the local runner
- [x] Daytona sandbox with your own account and snapshot
- [ ] E2B

**How agents run**

- [x] Interactively in chat
- [x] On a cron schedule
- [x] On connected application events through Composio
- [ ] Through a generic inbound webhook

**Inspection and improvement**

- [x] Tracing
- [x] Immutable version history
- [x] Token usage
- [x] Estimated model cost when recognized
- [x] Evaluations

## See what your agents did

Background agents can fail quietly. A run may finish while one action did nothing, used the wrong context, or cost more than expected.

Agenta traces each run so you can inspect the agent, model, and tool calls. You can compare the run with the agent version that produced it, review token usage and estimated cost, and use that evidence to improve the next version.

## Get started

### Agenta Cloud

The fastest way to try it. Free tier, no credit card.

[**Try Agenta Cloud**](https://cloud.agenta.ai?utm_source=github&utm_medium=referral&utm_campaign=readme)

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

For remote hosts and custom ports, see the [self-hosting guide](https://agenta.ai/docs/self-host/quick-start?utm_source=github&utm_medium=referral&utm_campaign=readme) and [remote deployment guide](https://agenta.ai/docs/self-host/guides/deploy-remotely?utm_source=github&utm_medium=referral&utm_campaign=readme).

## What's rough today

Agenta is early. We would rather state the limits clearly than let people discover them after installation.

- The supported harness and model matrix is still narrow.
- Subscription authentication requires manual self-hosted configuration.
- Composio integrations require the operator's own Composio key.
- User-configured MCP servers use HTTP transport and currently work with the Claude harness only.
- The default local runner executes inside the runner container and is not a tenant isolation boundary. Daytona provides a separate sandbox option.
- Templates and tutorials are still limited.

## What comes next

- More harnesses and model options
- Tested templates for go-to-market, engineering, support, and operations
- More tools, MCP servers, and application integrations
- Better background operation
- Easier team sharing and access control
- Better ways to improve agents from traces and evaluations

Want a specific harness, model, integration, or template? [Open a GitHub Discussion](https://github.com/Agenta-AI/agenta/discussions) or contribute it.

## Help us build the template library

We have just built this version of Agenta, and the template library is still thin. We would like to learn from the agents you are building and the work you want to automate.

You can help by:

- Sharing a real agent and the job it performs
- Contributing a template or tutorial
- Reporting where installation or onboarding failed
- Requesting an integration or trigger
- Showing us a recurring task you want to automate

## Community

- [Documentation](https://agenta.ai/docs/?utm_source=github&utm_medium=referral&utm_campaign=readme): guides and API reference
- [Changelog](https://agenta.ai/docs/changelog?utm_source=github&utm_medium=referral&utm_campaign=readme): recent updates
- [Slack](https://join.slack.com/t/agenta-hq/shared_invite/zt-37pnbp5s6-mbBrPL863d_oLB61GSNFjw): ask questions and get help
- [Report a bug](https://github.com/Agenta-AI/agenta/issues)
- [Share an idea](https://github.com/Agenta-AI/agenta/discussions)
- [Contributing guide](CONTRIBUTING.md)

## Star Agenta

**Consider giving us a star!** It helps us grow our community and gets Agenta in front of more developers.

<p align="center">
    <a href="https://github.com/agenta-ai/agenta">
  <img width="300" alt="Star us" src="https://github.com/user-attachments/assets/2c8e580a-c930-4312-bf1b-08f631b41c62" />
    </a>
</p>

## Contributors ✨

<!-- ALL-CONTRIBUTORS-BADGE:START - Do not remove or modify this section -->
[![All Contributors](https://img.shields.io/badge/all_contributors-68-orange.svg?style=flat-square)](#contributors-)
<!-- ALL-CONTRIBUTORS-BADGE:END -->

Thanks goes to these wonderful people ([emoji key](https://allcontributors.org/docs/en/emoji-key)):

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore-start -->
<!-- markdownlint-disable -->
<table>
  <tbody>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/SamMethnani"><img src="https://avatars.githubusercontent.com/u/57623556?v=4?s=100" width="100px;" alt="Sameh Methnani"/><br /><sub><b>Sameh Methnani</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=SamMethnani" title="Code">💻</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=SamMethnani" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/suadsuljovic"><img src="https://avatars.githubusercontent.com/u/8658374?v=4?s=100" width="100px;" alt="Suad Suljovic"/><br /><sub><b>Suad Suljovic</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=suadsuljovic" title="Code">💻</a> <a href="#design-suadsuljovic" title="Design">🎨</a> <a href="#mentoring-suadsuljovic" title="Mentoring">🧑‍🏫</a> <a href="https://github.com/Agenta-AI/agenta/pulls?q=is%3Apr+reviewed-by%3Asuadsuljovic" title="Reviewed Pull Requests">👀</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/burtenshaw"><img src="https://avatars.githubusercontent.com/u/19620375?v=4?s=100" width="100px;" alt="burtenshaw"/><br /><sub><b>burtenshaw</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=burtenshaw" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://abram.tech"><img src="https://avatars.githubusercontent.com/u/55067204?v=4?s=100" width="100px;" alt="Abram"/><br /><sub><b>Abram</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=aybruhm" title="Code">💻</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=aybruhm" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://israelabebe.com"><img src="https://avatars.githubusercontent.com/u/7479824?v=4?s=100" width="100px;" alt="Israel Abebe"/><br /><sub><b>Israel Abebe</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3Avernu" title="Bug reports">🐛</a> <a href="#design-vernu" title="Design">🎨</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=vernu" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/SohaibAnwaar"><img src="https://avatars.githubusercontent.com/u/29427728?v=4?s=100" width="100px;" alt="Master X"/><br /><sub><b>Master X</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=SohaibAnwaar" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://main-portfolio-26wv6oglp-witehound.vercel.app/"><img src="https://avatars.githubusercontent.com/u/26417477?v=4?s=100" width="100px;" alt="corinthian"/><br /><sub><b>corinthian</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=witehound" title="Code">💻</a> <a href="#design-witehound" title="Design">🎨</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Pajko97"><img src="https://avatars.githubusercontent.com/u/25198892?v=4?s=100" width="100px;" alt="Pavle Janjusevic"/><br /><sub><b>Pavle Janjusevic</b></sub></a><br /><a href="#infra-Pajko97" title="Infrastructure (Hosting, Build-Tools, etc)">🚇</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://kaosiso-ezealigo.netlify.app"><img src="https://avatars.githubusercontent.com/u/99529776?v=4?s=100" width="100px;" alt="Kaosi Ezealigo"/><br /><sub><b>Kaosi Ezealigo</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3Abekossy" title="Bug reports">🐛</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=bekossy" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/albnunes"><img src="https://avatars.githubusercontent.com/u/46302915?v=4?s=100" width="100px;" alt="Alberto Nunes"/><br /><sub><b>Alberto Nunes</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3Aalbnunes" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://www.linkedin.com/in/mohammed-maaz-6290b0116/"><img src="https://avatars.githubusercontent.com/u/17180132?v=4?s=100" width="100px;" alt="Maaz Bin Khawar"/><br /><sub><b>Maaz Bin Khawar</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=MohammedMaaz" title="Code">💻</a> <a href="https://github.com/Agenta-AI/agenta/pulls?q=is%3Apr+reviewed-by%3AMohammedMaaz" title="Reviewed Pull Requests">👀</a> <a href="#mentoring-MohammedMaaz" title="Mentoring">🧑‍🏫</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/devgenix"><img src="https://avatars.githubusercontent.com/u/56418363?v=4?s=100" width="100px;" alt="Nehemiah Onyekachukwu Emmanuel"/><br /><sub><b>Nehemiah Onyekachukwu Emmanuel</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=devgenix" title="Code">💻</a> <a href="#example-devgenix" title="Examples">💡</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=devgenix" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/philipokiokio"><img src="https://avatars.githubusercontent.com/u/55271518?v=4?s=100" width="100px;" alt="Philip Okiokio"/><br /><sub><b>Philip Okiokio</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=philipokiokio" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://sweetdevil144.github.io/My-Website/"><img src="https://avatars.githubusercontent.com/u/117591942?v=4?s=100" width="100px;" alt="Abhinav Pandey"/><br /><sub><b>Abhinav Pandey</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=Sweetdevil144" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/RamchandraWarang9822"><img src="https://avatars.githubusercontent.com/u/92023869?v=4?s=100" width="100px;" alt="Ramchandra Warang"/><br /><sub><b>Ramchandra Warang</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=RamchandraWarang9822" title="Code">💻</a> <a href="https://github.com/Agenta-AI/agenta/issues?q=author%3ARamchandraWarang9822" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/lazyfuhrer"><img src="https://avatars.githubusercontent.com/u/64888892?v=4?s=100" width="100px;" alt="Biswarghya Biswas"/><br /><sub><b>Biswarghya Biswas</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=lazyfuhrer" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/okieLoki"><img src="https://avatars.githubusercontent.com/u/96105929?v=4?s=100" width="100px;" alt="Uddeepta Raaj Kashyap"/><br /><sub><b>Uddeepta Raaj Kashyap</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=okieLoki" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://www.linkedin.com/in/nayeem-abdullah-317098141"><img src="https://avatars.githubusercontent.com/u/32274108?v=4?s=100" width="100px;" alt="Nayeem Abdullah"/><br /><sub><b>Nayeem Abdullah</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=nayeem01" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/kangsuhyun-yanolja"><img src="https://avatars.githubusercontent.com/u/124246127?v=4?s=100" width="100px;" alt="Kang Suhyun"/><br /><sub><b>Kang Suhyun</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=kangsuhyun-yanolja" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/yeokyeong-yanolja"><img src="https://avatars.githubusercontent.com/u/128676129?v=4?s=100" width="100px;" alt="Yoon"/><br /><sub><b>Yoon</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=yeokyeong-yanolja" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://mrkirthi24.netlify.app/"><img src="https://avatars.githubusercontent.com/u/53830546?v=4?s=100" width="100px;" alt="Kirthi Bagrecha Jain"/><br /><sub><b>Kirthi Bagrecha Jain</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=mrkirthi-24" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/navdeep1840"><img src="https://avatars.githubusercontent.com/u/80774259?v=4?s=100" width="100px;" alt="Navdeep"/><br /><sub><b>Navdeep</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=navdeep1840" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://www.linkedin.com/in/rhythm-sharma-708a421a8/"><img src="https://avatars.githubusercontent.com/u/64489317?v=4?s=100" width="100px;" alt="Rhythm Sharma"/><br /><sub><b>Rhythm Sharma</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=Rhythm-08" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://osinachi.me"><img src="https://avatars.githubusercontent.com/u/40396070?v=4?s=100" width="100px;" alt="Osinachi Chukwujama "/><br /><sub><b>Osinachi Chukwujama </b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=vicradon" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://liduos.com/"><img src="https://avatars.githubusercontent.com/u/47264881?v=4?s=100" width="100px;" alt="莫尔索"/><br /><sub><b>莫尔索</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=morsoli" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://luccithedev.com"><img src="https://avatars.githubusercontent.com/u/22600781?v=4?s=100" width="100px;" alt="Agunbiade Adedeji"/><br /><sub><b>Agunbiade Adedeji</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=dejongbaba" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://techemmy.github.io/"><img src="https://avatars.githubusercontent.com/u/43725109?v=4?s=100" width="100px;" alt="Emmanuel Oloyede"/><br /><sub><b>Emmanuel Oloyede</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=techemmy" title="Code">💻</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=techemmy" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Dhaneshwarguiyan"><img src="https://avatars.githubusercontent.com/u/116065351?v=4?s=100" width="100px;" alt="Dhaneshwarguiyan"/><br /><sub><b>Dhaneshwarguiyan</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=Dhaneshwarguiyan" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/PentesterPriyanshu"><img src="https://avatars.githubusercontent.com/u/98478305?v=4?s=100" width="100px;" alt="Priyanshu Prajapati"/><br /><sub><b>Priyanshu Prajapati</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=PentesterPriyanshu" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://venkataravitejagullapudi.github.io/"><img src="https://avatars.githubusercontent.com/u/70102577?v=4?s=100" width="100px;" alt="Raviteja"/><br /><sub><b>Raviteja</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=VenkataRavitejaGullapudi" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/ArijitCloud"><img src="https://avatars.githubusercontent.com/u/81144422?v=4?s=100" width="100px;" alt="Arijit"/><br /><sub><b>Arijit</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=ArijitCloud" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Yachika9925"><img src="https://avatars.githubusercontent.com/u/147185379?v=4?s=100" width="100px;" alt="Yachika9925"/><br /><sub><b>Yachika9925</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=Yachika9925" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Dhoni77"><img src="https://avatars.githubusercontent.com/u/53973174?v=4?s=100" width="100px;" alt="Aldrin"/><br /><sub><b>Aldrin</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=Dhoni77" title="Tests">⚠️</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/seungduk-yanolja"><img src="https://avatars.githubusercontent.com/u/115020208?v=4?s=100" width="100px;" alt="seungduk.kim.2304"/><br /><sub><b>seungduk.kim.2304</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=seungduk-yanolja" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://dandrei.com/"><img src="https://avatars.githubusercontent.com/u/59015981?v=4?s=100" width="100px;" alt="Andrei Dragomir"/><br /><sub><b>Andrei Dragomir</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=andreiwebdev" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://diegolikescode.me/"><img src="https://avatars.githubusercontent.com/u/57499868?v=4?s=100" width="100px;" alt="diego"/><br /><sub><b>diego</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=diegolikescode" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/brockWith"><img src="https://avatars.githubusercontent.com/u/105627491?v=4?s=100" width="100px;" alt="brockWith"/><br /><sub><b>brockWith</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=brockWith" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://denniszelada.wordpress.com/"><img src="https://avatars.githubusercontent.com/u/219311?v=4?s=100" width="100px;" alt="Dennis Zelada"/><br /><sub><b>Dennis Zelada</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=denniszelada" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/romainrbr"><img src="https://avatars.githubusercontent.com/u/10381609?v=4?s=100" width="100px;" alt="Romain Brucker"/><br /><sub><b>Romain Brucker</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=romainrbr" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="http://heonheo.com"><img src="https://avatars.githubusercontent.com/u/76820291?v=4?s=100" width="100px;" alt="Heon Heo"/><br /><sub><b>Heon Heo</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=HeonHeo23" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Drewski2222"><img src="https://avatars.githubusercontent.com/u/39228951?v=4?s=100" width="100px;" alt="Drew Reisner"/><br /><sub><b>Drew Reisner</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=Drewski2222" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://speakerdeck.com/eltociear"><img src="https://avatars.githubusercontent.com/u/22633385?v=4?s=100" width="100px;" alt="Ikko Eltociear Ashimine"/><br /><sub><b>Ikko Eltociear Ashimine</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=eltociear" title="Documentation">📖</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/vishalvanpariya"><img src="https://avatars.githubusercontent.com/u/27823328?v=4?s=100" width="100px;" alt="Vishal Vanpariya"/><br /><sub><b>Vishal Vanpariya</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=vishalvanpariya" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/youcefs21"><img src="https://avatars.githubusercontent.com/u/34604972?v=4?s=100" width="100px;" alt="Youcef Boumar"/><br /><sub><b>Youcef Boumar</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=youcefs21" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/LucasTrg"><img src="https://avatars.githubusercontent.com/u/47852577?v=4?s=100" width="100px;" alt="LucasTrg"/><br /><sub><b>LucasTrg</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=LucasTrg" title="Code">💻</a> <a href="https://github.com/Agenta-AI/agenta/issues?q=author%3ALucasTrg" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://ashrafchowdury.me"><img src="https://avatars.githubusercontent.com/u/87828904?v=4?s=100" width="100px;" alt="Ashraf Chowdury"/><br /><sub><b>Ashraf Chowdury</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3Aashrafchowdury" title="Bug reports">🐛</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=ashrafchowdury" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/jp-agenta"><img src="https://avatars.githubusercontent.com/u/174311389?v=4?s=100" width="100px;" alt="jp-agenta"/><br /><sub><b>jp-agenta</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=jp-agenta" title="Code">💻</a> <a href="https://github.com/Agenta-AI/agenta/issues?q=author%3Ajp-agenta" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://mrunhap.github.io"><img src="https://avatars.githubusercontent.com/u/24653356?v=4?s=100" width="100px;" alt="Mr Unhappy"/><br /><sub><b>Mr Unhappy</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3Amrunhap" title="Bug reports">🐛</a> <a href="#infra-mrunhap" title="Infrastructure (Hosting, Build-Tools, etc)">🚇</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/morenobonaventura"><img src="https://avatars.githubusercontent.com/u/2118854?v=4?s=100" width="100px;" alt="Moreno Bonaventura"/><br /><sub><b>Moreno Bonaventura</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3Amorenobonaventura" title="Bug reports">🐛</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://ikazoy.me/"><img src="https://avatars.githubusercontent.com/u/385109?v=4?s=100" width="100px;" alt="Yoshiki Ozaki"/><br /><sub><b>Yoshiki Ozaki</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3Aikazoy" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/ahmed-agenta"><img src="https://avatars.githubusercontent.com/u/194256084?v=4?s=100" width="100px;" alt="ahmed-agenta"/><br /><sub><b>ahmed-agenta</b></sub></a><br /><a href="#design-ahmed-agenta" title="Design">🎨</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/VahantSharma"><img src="https://avatars.githubusercontent.com/u/172914890?v=4?s=100" width="100px;" alt="Vahant Sharma"/><br /><sub><b>Vahant Sharma</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=VahantSharma" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/muzman123"><img src="https://avatars.githubusercontent.com/u/66068301?v=4?s=100" width="100px;" alt="Muhammad Muzammil"/><br /><sub><b>Muhammad Muzammil</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=muzman123" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/CyrusNamjoo"><img src="https://avatars.githubusercontent.com/u/209579763?v=4?s=100" width="100px;" alt="Sirous Namjoo"/><br /><sub><b>Sirous Namjoo</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=CyrusNamjoo" title="Documentation">📖</a> <a href="#example-CyrusNamjoo" title="Examples">💡</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/adityadewan22-hub"><img src="https://avatars.githubusercontent.com/u/225586510?v=4?s=100" width="100px;" alt="adityadewan22-hub"/><br /><sub><b>adityadewan22-hub</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=adityadewan22-hub" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/majiayu000"><img src="https://avatars.githubusercontent.com/u/19658300?v=4?s=100" width="100px;" alt="lif"/><br /><sub><b>lif</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=majiayu000" title="Code">💻</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="http://karimkohel.com"><img src="https://avatars.githubusercontent.com/u/46066647?v=4?s=100" width="100px;" alt="karim kohel"/><br /><sub><b>karim kohel</b></sub></a><br /><a href="#example-karimkohel" title="Examples">💡</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=karimkohel" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Vishesh-Paliwal"><img src="https://avatars.githubusercontent.com/u/142072830?v=4?s=100" width="100px;" alt="Vishesh Paliwal"/><br /><sub><b>Vishesh Paliwal</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=Vishesh-Paliwal" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/aviu16"><img src="https://avatars.githubusercontent.com/u/162624394?v=4?s=100" width="100px;" alt="Eve"/><br /><sub><b>Eve</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=aviu16" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://endoze.github.io"><img src="https://avatars.githubusercontent.com/u/997161?v=4?s=100" width="100px;" alt="Endoze"/><br /><sub><b>Endoze</b></sub></a><br /><a href="#infra-endoze" title="Infrastructure (Hosting, Build-Tools, etc)">🚇</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=endoze" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/anshk8"><img src="https://avatars.githubusercontent.com/u/141085661?v=4?s=100" width="100px;" alt="Ansh Kakkar"/><br /><sub><b>Ansh Kakkar</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3Aanshk8" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Devarsh05"><img src="https://avatars.githubusercontent.com/u/116822773?v=4?s=100" width="100px;" alt="Devarsh Prajapati"/><br /><sub><b>Devarsh Prajapati</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3ADevarsh05" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/axelray-dev"><img src="https://avatars.githubusercontent.com/u/110029405?v=4?s=100" width="100px;" alt="AxelRay"/><br /><sub><b>AxelRay</b></sub></a><br /><a href="#platform-axelray-dev" title="Packaging/porting to new platform">📦</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=axelray-dev" title="Documentation">📖</a></td>
    </tr>
    <tr>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Sanket2329"><img src="https://avatars.githubusercontent.com/u/196506711?v=4?s=100" width="100px;" alt="Sanket Shakya"/><br /><sub><b>Sanket Shakya</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=Sanket2329" title="Code">💻</a> <a href="#platform-Sanket2329" title="Packaging/porting to new platform">📦</a> <a href="https://github.com/Agenta-AI/agenta/commits?author=Sanket2329" title="Documentation">📖</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/unfitcoder101"><img src="https://avatars.githubusercontent.com/u/175036458?v=4?s=100" width="100px;" alt="unfitcoder101"/><br /><sub><b>unfitcoder101</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3Aunfitcoder101" title="Bug reports">🐛</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Shunmuka"><img src="https://avatars.githubusercontent.com/u/137101604?v=4?s=100" width="100px;" alt="Shunmuka Valsa"/><br /><sub><b>Shunmuka Valsa</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=Shunmuka" title="Code">💻</a> <a href="#platform-Shunmuka" title="Packaging/porting to new platform">📦</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/NamHT4Devlop"><img src="https://avatars.githubusercontent.com/u/122743792?v=4?s=100" width="100px;" alt="Hồ Trung Nam"/><br /><sub><b>Hồ Trung Nam</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/commits?author=NamHT4Devlop" title="Code">💻</a></td>
      <td align="center" valign="top" width="14.28%"><a href="https://github.com/Koushik-Salammagari"><img src="https://avatars.githubusercontent.com/u/138836560?v=4?s=100" width="100px;" alt="Koushik-Salammagari"/><br /><sub><b>Koushik-Salammagari</b></sub></a><br /><a href="https://github.com/Agenta-AI/agenta/issues?q=author%3AKoushik-Salammagari" title="Bug reports">🐛</a></td>
    </tr>
  </tbody>
</table>

<!-- markdownlint-restore -->
<!-- prettier-ignore-end -->

<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/all-contributors/all-contributors) specification. Contributions of any kind are welcome!

## License

MIT.
