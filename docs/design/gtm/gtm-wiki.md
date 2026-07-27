# Agenta Go-to-Market Wiki

_Last updated: 2026-07-12 — instrumentation, assets, team, and experiment framing_

## Purpose

This is the living source for organizing Agenta's GTM. It preserves founder context, distinguishes observations from hypotheses and decisions, and will gradually become an actionable launch and learning plan.

## Operating context

- Today is Sunday, 12 July 2026.
- The first public launch is planned for Tuesday, 14 July 2026.
- This is the first of many launches during an intensive 2–3 month GTM period.
- The immediate process is brain dump first, then synthesis, prioritization, milestones, and experiments.
- This file is updated after every relevant conversation turn.

## Goal

Find the initial segment for which the product already creates strong value and which can support rapid growth. Then optimize the product and messaging for that segment before expanding to adjacent segments.

The segment is not yet known. GTM is therefore a discovery process, not just a distribution plan.

## Product origin and founder evidence

- The product is an MVP built primarily for the founder's own needs rather than for a validated segment.
- The founder wants it to become a primary everyday place to work.
- In practice, the founder particularly values talking with specialized agents, creating artifacts, retaining files and context, and sharing chats with a team.
- Chat and artifacts were not originally considered the main value proposition, but hands-on use has made them feel increasingly important.
- For implementation-heavy engineering work, the founder currently prefers Claude and engineering-specific workflows. This is evidence that Agenta is not currently the strongest complete environment for that job.

## Product: factual capability map

### Conversational work

- ChatGPT-like conversations with configurable, specialized agents.
- Agents can help perform work, explain themselves, and help improve their own setup.
- Sessions can retain in-scope memory and durable files that users can inspect.
- The UI supports files and questionnaires, with richer artifacts and generative UI as part of the product direction.
- Chats and work can be shared within a team.

### Agent configuration

Agents can be configured with:

- Instructions.
- Tools, including shell/coding tools and external integrations through Composio.
- MCP servers.
- Skills.
- Harnesses (including Claude, with Codex contemplated for the future).
- General and granular permissions.
- Scheduled and event-based triggers.

### Interactive and background execution

- An agent can work interactively through chat.
- It can also run when triggered by time or external events.
- A user can therefore develop work conversationally and potentially turn it into recurring background work.

### Build kit and self-configuration

- The Playground build kit lets an agent configure itself using platform tools.
- The intended experience is that a user describes what they want and the agent helps become the appropriate collaborator, workflow, or background automation.

### Platform, governance, and portability

- Agent configuration is versioned and earlier revisions can be inspected or restored.
- The platform retains tracing, observability, annotations, evaluations, and role-based access control from the previous Agenta product.
- Agents and skills can be shared across teams subject to access controls.
- Agenta is open source and self-hostable.
- Users can use their own model subscriptions or API keys.
- The configuration is general enough in principle to be loaded and run elsewhere, although the founder currently sees value in Agenta itself as the place to work.

## Product boundaries and constraints observed so far

- Flexibility is intentional; Agenta does not force work into rigid predefined workflow “tunnels.”
- Getting started should be conversational and accessible to somewhat technical users without requiring deep platform knowledge.
- Advanced use may still require learning or training.
- The breadth of capability creates a risk that the product is hard to explain without concrete workflows or templates.
- Tracing, evaluations, and orchestration are capabilities, but are not believed to be the primary user-facing value today.

## Candidate segments

These are unvalidated possibilities, not chosen ICPs:

1. Startup founders/CEOs seeking a flexible AI work and automation environment.
2. Freelancers or automation consultants currently building workflows with tools such as n8n.
3. Tinkerers automating personal workflows and services such as Telegram.
4. Engineers automating bounded engineering-adjacent work.
5. Teams/businesses collaborating around shared agents, skills, chats, and workflows.
6. Non-technical users who want useful automation but do not know how to build it directly.

### Engineering segment: current evidence

Potential jobs include QA review, support cleanup, bounded task generation, and design-tool workflows. However:

- Engineers often need code visibility and strong version-history workflows.
- They may need relatively few integrations compared with general business workflows.
- They already work in specialized environments where an automation may ultimately belong.
- Agenta may be useful for bounded engineering automations without being the best full engineering workspace.

## Value proposition: current founder view

The value is the work users can accomplish and automate—not “agent orchestration.” That phrase is engineering language, unclear to a general user, and does not express an outcome.

There are two broad modes of value:

1. **Interactive work:** an agent uses relevant skills, tools, integrations, context, and conversation to help complete work and create artifacts.
2. **Recurring work:** an agent handles repeat work on a schedule or in response to an event.

A potentially important product progression is:

**conversation → useful artifact/output → reusable agent workflow → triggered background work**

This progression is a hypothesis, not yet a validated unique selling proposition.

## Why templates may matter

Templates could:

- Explain the product through concrete jobs and outcomes.
- Reduce blank-page friction and time to first value.
- Package the product for different segment hypotheses.
- Let GTM test demand by use case before choosing an ICP.
- Help users progress from interactive work to repeat automation.

No specific template set has yet been selected.

## Positioning hypotheses—not decisions

Possible category descriptions:

- AI automation platform.
- Conversational agent workspace.
- Agent workspace.
- Agent build kit.

Possible message territories:

- Start with a conversation; turn it into work that runs itself.
- Automate the recurring work that consumes your week.
- Give an AI agent the skills and tools to handle a real workflow.
- Chat, create the output, then automate the workflow.

The final category, message, ICP, and USP remain open.

## What is explicitly not decided

- Initial ICP or buyer.
- Initial use case or template wedge.
- Product category.
- Primary value proposition or USP.
- Tuesday's launch scope, audience, channel, offer, CTA, or success criterion.
- Pricing or packaging.
- The milestones for the first week, first month, or 2–3 month period.

## Missing context to collect

### Product readiness

- What can a new user reliably do today from signup to completed outcome?
- Which capabilities are live, partially working, future-facing, or internal-only?
- Where does onboarding fail or require founder assistance?
- What are the known reliability, UX, security, and deployment constraints?

### Existing users and evidence

- Who has used the new product besides the founder?
- What did they attempt, complete, repeat, or abandon?
- Which moments caused delight or confusion?
- Are there users, waitlists, former Agenta users, community members, or design partners available for discovery?
- What usage, retention, trace, or qualitative data already exists?

### Founder workflow evidence

- Which exact workflows does the founder use today?
- How frequently, and what did they replace?
- What outcomes or artifacts are produced?
- Which workflow creates enough value that losing it would hurt?
- How much setup or intervention is currently required?

### Market and alternatives

- Which products do prospective users use instead for each job: ChatGPT/Claude, n8n, Gumloop, cowork products, coding agents, human assistants, scripts, or manual work?
- Why would a user switch, and what would prevent switching?
- Which parts of the proposed experience are actually differentiated versus simply expected?

### Distribution assets and constraints

- Founder audience, company audience, email list, former-user base, communities, partners, and available launch channels.
- Current website, domain, demo, screenshots, videos, templates, docs, onboarding, and signup flow.
- Team capacity for product fixes, content, onboarding calls, and support over the next two weeks.
- Budget and appetite for hands-on founder-led onboarding.

### Business constraints

- Desired business model and whether the near-term objective is learning, usage, revenue, fundraising evidence, or community growth.
- Pricing assumptions, hosting economics, model costs, and self-hosted versus cloud priorities.
- Geographic, regulatory, security, or procurement constraints.

### Tuesday deadline

- Why Tuesday is the chosen date.
- What “launch” means operationally.
- What must be true for it to proceed.
- What audience can realistically be reached by then.
- What signal the launch should produce.

## Working method from here

Continue separating:

- **Context/evidence:** what exists or has happened.
- **Hypotheses:** what we believe may be true.
- **Decisions:** choices we intentionally make.
- **Experiments:** actions designed to resolve uncertainty.
- **Results:** evidence produced by those experiments.


## Context update: readiness, audience, urgency, and launch model

### Product readiness as of 12 July 2026

Working now, though with rough edges:

- Creating a new agent.
- Chatting with an agent.
- Changing agent configuration.
- Adding tools.
- Adding triggers.
- Running the core conversational/configuration loop demonstrated in this GTM project.

Current limitations:

- The experience is not polished or fully reliable.
- Bugs remain and new users may encounter failures.
- Documentation and videos are not yet sufficient to eliminate assistance.
- Broader behavioral and qualitative usage evidence does not yet exist.
- The product is only now reaching initial readiness; the live self-hosted release is expected by the end of Monday or on Tuesday.

This readiness level makes tolerant, self-directed early adopters more appropriate than mainstream or enterprise users.

### Product and distribution model

- Agenta is open source and MIT-licensed.
- The immediate product to drive adoption for is the self-hosted version.
- The open-source community is central to both product development and GTM.
- Desired early users will try imperfect software, investigate problems, create GitHub issues, discuss failures in Slack, and potentially contribute tests or fixes.
- The team will create launch materials now, including website updates, documentation, self-hosting documentation, and videos.

### Evidence available today

Current users/testers are limited to:

- The founder.
- The Agenta team.
- Friends and family.

There are no meaningful behavioral, retention, or qualitative datasets yet. The founder has only just begun using the new Agenta in a real workflow; this GTM agent/chat is the first substantive workflow and real project. Therefore, claims about repeat usage, strongest workflows, or retention remain premature.

### Existing reach

- Approximately 20,000 people are on Agenta's mailing list.
- The founder has approximately 6,000 LinkedIn followers.
- There are no current design partners, and enterprise design-partner selling is not a priority for this phase.
- Reddit, X/Twitter, GitHub, Slack, self-hosting communities, direct outreach, and the existing audience are possible distribution surfaces.
- Tuesday is not intended to be a maximal LinkedIn/Product Hunt-style launch.

The size of the existing audiences is known; their fit with the pivoted product is not.

### Immediate business constraint

- The company expects to need fundraising in roughly two months as runway becomes constrained.
- The near-term objective is not revenue.
- The overriding objective is adoption: self-hosted installs, usage, community participation, learning, and open-source growth.
- Product and GTM work should avoid revenue-oriented distractions during this phase unless the constraint changes.
- The ambition is for Agenta to become the default open-source alternative to products such as ChatGPT-style and Claude-style agent/work experiences for creating and using agents.

### Desired early-adopter characteristics

The initial users should ideally be:

- Curious tinkerers and self-directed learners.
- Comfortable with technical concepts and self-hosting.
- Tolerant of bugs and rough edges.
- Able to create many agents or identify many workflows.
- Motivated to learn about agents and AI automation.
- Willing to report issues and participate publicly in a community.
- Likely to talk about Agenta and show what they build.
- Part of a segment that can expand beyond a tiny niche over time.

Candidate communities within this profile include:

- Early-career engineers moving into AI.
- AI/automation tinkerers.
- Technical startup founders.
- Potentially automation freelancers, although their early-adopter fit is less certain.

Long-term, enterprise and business use may become attractive because Agenta already has foundations such as RBAC and SSO. However, current reliability and quality are not sufficient for that market, and enterprise is not the immediate GTM focus.

### Competitive context: initial founder view

Compared with general AI chat/work products such as ChatGPT and Claude:

- Agenta offers greater configurability and flexibility.
- Users can switch models and harnesses and potentially use self-hosted models.
- Agenta is open source and self-hostable.
- It supports explicit agent workflows, tools, skills, triggers, and inspectable configuration.

Compared with workflow products such as Gumloop and n8n:

- Agenta is more conversational and agent-centric.
- It exposes more of the internal configuration and is more suitable for technically curious users who want to manipulate the underlying system.
- It is open source and MIT-licensed.
- It combines interactive chat/artifact work with triggered workflows.

These are initial comparison points, not yet verified differentiation. Some competitors are moving toward similar agent experiences, so several advantages may be temporary. The durable importance of openness, licensing, self-hosting, model/harness choice, and community still needs validation.

### Why launch now

- The pivot began approximately one month ago.
- A working product now exists and further learning requires putting it in users' hands.
- Feedback and real-world use are more valuable at this stage than waiting for polish.
- Tuesday should be treated as the start of continuous exposure and iteration, not the definitive launch.

### Launch model

The intended model is repeated launching rather than a single event:

- Begin on Tuesday with a focused early-adopter/self-hosted release.
- Use targeted communities, outreach, and potentially small paid-message tests rather than spending all attention on a broad splash.
- Continuously test segments, use cases, templates, messaging, positioning, and value propositions.
- Ship product, documentation, templates, and content repeatedly over the following weeks.
- Build enough exposure that tests produce meaningful signals; one isolated launch is insufficient.
- Aim to accumulate learning and momentum toward a larger launch in roughly 4–6 weeks.
- Relaunching several times is acceptable and expected.

### Primary learning loop

The near-term GTM loop is:

1. Put the self-hosted product in the hands of suitable early adopters.
2. Observe who installs it, what they try to build, what works, where they fail, and whether they return.
3. Gather feedback and community participation through GitHub and Slack.
4. Segment users by persona, workflow, behavior, and value received.
5. Create and distribute workflow templates and messaging for promising segments.
6. Improve the product rapidly around demonstrated usage.
7. Repeat launches and experiments until a segment/message/workflow combination shows strong adoption potential.

### Metrics not yet defined

“Adoption” is the priority, but its operational definition remains open. Candidate measures to evaluate later include:

- Self-hosted installations and successful first runs.
- Agents created and meaningfully used.
- Workflows completed and triggered repeatedly.
- Return usage.
- GitHub stars, issues, contributors, and pull requests.
- Slack joins and useful participation.
- Templates installed, used, or shared.
- Users publicly sharing what they built.

No target numbers or primary metric have been selected yet.


## Context update: instrumentation, assets, audience, and team

### Instrumentation

- The product is instrumented with PostHog, including the open-source product.
- Additional activities/actions can be added as needed.
- The team can analyze retention and churn.
- This means segment experiments can be evaluated using behavior after acquisition, not only clicks or stated interest.

### Templates

- The founder reports that 23 templates already exist in the product.
- The currently injected build-kit template index exposes 28 playbooks across engineering, support, sales, monitoring, knowledge, and operations. These may not be the same set as the founder's 23 product templates, so the exact product inventory still needs reconciliation rather than assuming they are identical.
- The visible playbook categories suggest broad workflow coverage, but coverage itself does not establish which templates are ready, compelling, or suitable for the initial audience.

### Existing community and audience

- The GitHub repository has approximately 4,000 stars.
- Slack membership was stated as approximately “F500,” likely around 500, but the exact number should be confirmed.
- The 20,000-person mailing list consists of people who joined the previous Agenta product, generally AI engineers and AI product people.
- The founder estimates roughly 20% of the mailing-list audience may fit the new product, implying a plausible reachable pool of about 4,000 people. This is an estimate, not observed conversion data.

### Team capacity

The team is five core people plus two juniors:

- Founder/CEO, currently central to product and shifting substantial focus toward GTM.
- CTO.
- Product engineer.
- Another main engineer.
- One additional core team member (role not yet specified).
- Two juniors.

The team is small, and the founder is currently the main person focused heavily on product and soon GTM. Experiments must therefore be fast, reusable, instrumented, and light on bespoke support.

### Assets and Tuesday readiness

- A designed website and current version already exist; copy and cleanup are needed.
- GitHub exists and has meaningful existing attention; repository messaging needs updating.
- Documentation is outdated but can be revised.
- Tuesday work is focused on bug cleanup and producing/updating necessary artifacts rather than building a wholly new product.

## Initial segment hypotheses for testing

These hypotheses are deliberately narrower than “everyone who can automate.” All target open-source-friendly early adopters.

### H1 — Technical founders and small startup operators

**Situation:** They personally span product, engineering, support, and GTM and repeatedly stitch together AI tools to do work.

**Why Agenta may fit:** The founder is a member of this segment; they have many cross-functional workflows, tolerate technical products, value flexible chat plus automation, and can become visible advocates.

**Risk:** “Founder” is broad, pain may be diffuse, and many already live in ChatGPT/Claude plus scripts.

### H2 — AI engineers and AI product builders who want internal automations outside the coding loop

**Situation:** They understand agents and want to automate support, QA, research, reporting, triage, or internal operations without building and maintaining a custom system.

**Why Agenta may fit:** Strong overlap with the existing mailing list and GitHub audience; high ability to self-host, debug, report issues, and appreciate model/harness choice and inspectable configuration.

**Risk:** They may prefer building directly or expect a developer-first code environment Agenta is not optimized to replace.

### H3 — AI automation tinkerers and aspiring agent builders

**Situation:** They actively experiment with n8n, MCP, local/self-hosted AI, Telegram bots, and new agent frameworks, and want a powerful environment they can inspect and modify.

**Why Agenta may fit:** High tolerance for rough edges, strong open-source affinity, many experiments, and likelihood of sharing what they build.

**Risk:** High curiosity may produce stars and one-time trials without durable workflow usage.

### H4 — Technical automation freelancers/consultants

**Situation:** They build multiple workflows for clients and need reusable agents, templates, integrations, and a shareable/self-hosted environment.

**Why Agenta may fit:** Repeated use cases, natural template creation, and potential distribution into client organizations.

**Risk:** Reliability, client management, deployment, and support needs may already exceed MVP readiness.

### Initial prioritization hypothesis

Start by comparing H1, H2, and H3. Keep H4 observational until evidence shows freelancers can use the current product without reliability or client-delivery blockers. Do not target generic non-technical users or enterprise teams in this phase.

## Proposed experiment system

### Experiment 0 — Establish cohort instrumentation

**Action:** Define acquisition source, declared persona, selected template/workflow, successful self-host, first agent created, first meaningful run, first triggered run, return usage, and community action as trackable properties/events.

**Learning:** Whether later experiments create installation, activation, and repeat behavior—not merely traffic or stars.

### Experiment 1 — Existing-audience segment invitation

**Action:** Send small, separately tagged invitations to matched slices of the existing mailing list/community. Use one message for AI engineers/product builders and another for technical founders/operators. Invite them to self-host, try one concrete workflow, and join/report through GitHub or Slack.

**Learning:** Which segment/message combination produces qualified clicks, successful installs, activation, return usage, and issue/community participation.

**Important:** Start with small batches rather than the entire 20,000-person list so messaging can improve without exhausting the audience.

### Experiment 2 — Persona-specific landing/message tests

**Action:** Create lightweight pages or page variants for H1, H2, and H3. Keep the product constant while changing the user, problem, example workflows, and proof. Drive small targeted traffic from Reddit/X or direct outreach.

**Learning:** Which framing attracts the right users and which workflow promise causes them to attempt installation.

### Experiment 3 — Template wedge test

**Action:** Select 2–3 ready templates for each leading segment, grouped around coherent jobs rather than platform categories. Present one recommended starting workflow per message/page rather than all templates at once.

**Learning:** Which jobs generate completed first runs, repeat use, and requests for adjacent workflows. This distinguishes curiosity about agents from demand for actual work.

### Experiment 4 — Concierge observation without enterprise selling

**Action:** Recruit 5–8 suitable early adopters across the leading hypotheses. Watch or closely review their installation and first workflow, then examine their traces/events and conduct short follow-ups.

**Learning:** Where activation fails, what users think the product is, which capabilities matter, and whether their second workflow emerges naturally.

### Experiment 5 — Community-channel probes

**Action:** Publish distinct, useful demonstrations in narrowly relevant communities: self-hosting/open-source, agent/MCP builders, technical founders, and automation tinkerers. Each post should show a concrete workflow and use tagged links.

**Learning:** Which communities yield activated users and contributors rather than only engagement. It also reveals the language each community uses for the problem.

### Experiment 6 — Conversation-to-automation proposition test

**Action:** Demonstrate the same task in two stages: accomplish it interactively in chat, then configure it to recur through a trigger. Compare this message against a simpler “open-source AI automation” message.

**Learning:** Whether the conversation → artifact → recurring work mechanism is understood, differentiated, and motivating.

### Experiment 7 — Open-source proposition test

**Action:** Compare emphasis on (a) open source/self-hosting and control, (b) model/harness flexibility, and (c) concrete workflow outcomes, while keeping audience and workflow stable.

**Learning:** Whether openness drives adoption itself, acts as trust/proof after an outcome attracts users, or appeals primarily to low-retention tinkerers.

## Research stream, not yet executed

Segment research on Reddit and X should run alongside launch experiments. Agents can collect posts, recurring complaints, existing tool stacks, vocabulary, and evidence of attempted workflows. Research should begin narrowly by segment/problem and preserve source links and exact language. It should inform hypotheses but not substitute for observed product use.

## Reddit research handoff

A separate research session will investigate Reddit evidence for the initial segment, workflow wedge, competitor gaps, and market language. It should map relevant subreddits, personas, concrete use cases, n8n and other competitor complaints, self-hosting motivations, and terms such as agents, background agents, loops, workflows, AI automation, harnesses, skills, and MCP.

The research must preserve source links, dates, short verbatim quotes, contradictory evidence, and a clear separation between observations and interpretations. Findings should inform segment and messaging experiments but should not be treated as final positioning decisions without product-usage evidence.

Handoff: `reddit-research-handoff.md`.


### Reddit research refinement: community-first mapping

The research should begin with a broad community map and then identify personas and interests within each community. Additional seed communities include AI_Automation, AI_Agents, LLMDevs, AutoGPT, ClaudeAI, OpenSourceAI, OpenAIDev, and especially CrewAI; exact subreddit names and activity must be verified.

Community discovery should expand through public cross-posts and aggregate participation patterns from contributors in high-signal threads. Profile review may be used to infer broad builder/persona archetypes and discover adjacent communities, but the output should not create personal dossiers or retain usernames.

For each community, research should cover size, activity, persona clusters, career/learning interests, what members build, verticals and application types, workflows/tasks, current excitement, recurring problems, and failures. Topic deep dives should prioritize experiences adjacent to Agenta: Claude's emerging work/agent capabilities, new n8n AI/chat features, chat-agent construction, CrewAI, background execution, loops, triggers, self-hosted agents, and durable interactive work.

## Planned GTM synthesis artifact

The next major artifact should be a downloadable, self-contained visual report connecting communities, personas, workflows/use cases, pains, competitive alternatives, and campaign experiments. It should include special attention to the self-hosting community and provide 2–3 positioning systems with corresponding UVP/USP, message variants, landing-page concepts, channels, and measurable tests. Work will begin after selecting and committing an appropriate set of reusable GTM and presentation skills.

## Interactive GTM opportunity map

A self-contained interactive report has been created at `agenta-gtm-opportunity-map.html`. It connects communities, personas, use cases, activation behaviors, positioning systems, campaign experiments, and source evidence. It includes a dedicated self-hosting strategy section and can be opened locally or printed to PDF.

## Three integrated GTM hypotheses

These are three coherent persona–position–message–channel systems to test independently. They are hypotheses, not final positioning. Do not combine all three into one homepage before behavioral evidence identifies the strongest segment × job × message combination.

### Hypothesis 1 — Technical founders: work that keeps running

#### Persona and situation

Technical founders and small-team operators handling GTM, research, support, product, and operations themselves. They already use ChatGPT or Claude for useful work, but the work remains in disconnected conversations and must be repeated manually.

#### Positioning

- **Category:** Open-source AI workspace.
- **Positioning statement:** For technical founders who repeatedly use AI across their company, Agenta is an open-source workspace that turns useful conversations into reusable workflows that keep running.
- **Value proposition:** Complete a job with an agent through conversation, preserve its artifacts and context, then trigger that workflow automatically.
- **Candidate USP/mechanism:** Conversation → artifact → reusable agent → recurring background work.

#### Messaging

- **Headline:** Start with a conversation. Turn it into work that runs itself.
- **Subhead:** Build an agent for a real founder workflow, refine it through chat, then schedule or trigger it—without giving up control of its tools, files, or configuration.
- **Pillars:** Specialized agents instead of restarting generic chats; durable project context and artifacts; automation for work worth repeating.

#### Initial workflow wedges

- Weekly market and competitor brief.
- Customer-feedback synthesis.
- Support triage.
- Launch planning and content production.
- Daily company or product briefing.

#### Channels

- Founder LinkedIn and X audiences.
- A small tagged cohort from the existing mailing list.
- Founder communities such as r/startups, r/SaaS, and r/SideProject, subject to community rules.
- AI-agent communities.
- Direct outreach to technically capable founders.

#### Experiment

Create three founder landing-page variants centered on market intelligence, support triage, and the broader founder-workspace promise. The campaign demonstration should show a conversation producing a market brief and then becoming a weekly triggered workflow.

- **Primary metric:** Meaningful first workflow completed.
- **Secondary metrics:** Trigger configured; second run completed; second related workflow created; return within seven days.
- **Decision rule:** Advance when founders naturally progress from one interactive workflow to repeat usage. Clicks alone do not validate the hypothesis.
- **Main risk:** “Technical founder” may be too broad and the pain may remain diffuse.

### Hypothesis 2 — Self-hosting and local-AI builders: own your agent workspace

#### Persona and situation

Self-hosting operators, local-AI builders, MCP tinkerers, and developers who prioritize control, privacy, portability, customization, and model choice. “Self-hoster” is a motivation layer rather than a complete ICP; the research must still identify the repeated workflow.

#### Positioning

- **Category:** Self-hosted agent workspace.
- **Positioning statement:** For technical users who want capable agents without surrendering control, Agenta is the open-source agent workspace that runs with their models, tools, data, and infrastructure.
- **Value proposition:** Create ChatGPT/Claude-like agent experiences that users can inspect, customize, self-host, and keep.
- **Candidate USP/mechanism:** MIT-licensed workspace + model/harness choice + inspectable configuration + interactive and triggered execution.

#### Messaging

- **Headline:** Your agents. Your models. Your infrastructure.
- **Subhead:** Build conversational and background agents with your own models, tools, MCP servers, files, and triggers—in an open-source workspace you control.
- **Pillars:** Self-host the complete workspace; switch models and tools; inspect configuration, permissions, versions, and traces; avoid dependence on one vendor.

#### Initial workflow wedges

- Local personal or work agent.
- Scheduled local intelligence agent.
- Telegram-connected agent.
- MCP-powered desktop or browser agent.
- Private team knowledge or operations agent.

#### Channels

- r/selfhosted and r/LocalLLaMA, subject to community rules.
- Open-source AI communities and directories.
- Hacker News.
- GitHub.
- Self-hosting newsletters and creators.
- X audiences focused on local AI and open source.

#### Experiment

Drive the same concrete self-hosted workflow with three message variants:

1. Ownership: “Your agents, models, and infrastructure.”
2. Privacy/control: “Your work does not have to live inside someone else’s AI.”
3. Workflow outcome: “Self-host an agent that prepares your daily brief.”

- **Primary metric:** Successful self-host followed by a meaningful workflow run.
- **Secondary metrics:** Repeated run; local-model connection; issue, contribution, or public workflow share.
- **Guardrail:** Stars, installs, and architecture discussion without meaningful repeated use.
- **Decision rule:** Use self-hosting as the primary position only if control-led acquisition creates retained workflows. Otherwise, treat open source as proof supporting an outcome-led position.
- **Main risk:** Architecture enthusiasm may attract high-curiosity, low-retention users.

### Hypothesis 3 — AI and automation builders: agent workflows you can operate

#### Persona and situation

AI engineers, AI product builders, advanced n8n users, and agent-framework experimenters building internal automations outside their primary coding loop. Agent demos are easy, but workflows that remain understandable, debuggable, and reliable are difficult. Visual automations become tangled while code-first frameworks require plumbing and maintenance.

#### Positioning

- **Category:** Open-source AI workflow platform.
- **Positioning statement:** For AI builders moving beyond demos, Agenta provides an interactive way to build, run, inspect, version, and improve agent workflows.
- **Value proposition:** Move from an agent prototype to repeatable work without assembling the entire configuration, execution, observability, and collaboration layer.
- **Candidate USP/mechanism:** Conversational building + explicit configuration + triggers + versioning + tracing/evaluations.

#### Messaging

- **Headline:** Build agent workflows you can actually operate.
- **Subhead:** Create the workflow through conversation, inspect every configuration, run it interactively or in the background, and improve it with traces and version history.
- **Pillars:** Build with chat without hiding the system; inspect tools, skills, permissions, and configuration; diagnose and revise failures; move between interactive and scheduled execution.

#### Initial workflow wedges

- QA review.
- Support triage.
- Feedback clustering.
- Monitoring and reporting.
- Research workflows.
- Agent evaluation and debugging.

#### Channels

- r/n8n, r/LangChain, r/crewai, and r/AI_Agents, subject to community rules.
- AI engineer and AI product segments of the existing mailing list.
- GitHub.
- Technical X audiences.
- Framework and MCP communities.

#### Experiment

Demonstrate a workflow failing to call a tool, then show Agenta exposing the trace and configuration, revising the agent, rerunning it, and versioning the fix. Test three landing-page messages:

1. “From agent demo to operable workflow.”
2. “When your n8n AI workflow becomes difficult to debug.”
3. “Build through conversation without hiding the configuration.”

- **Primary metric:** First successful meaningful workflow completed.
- **Secondary metrics:** Failed run successfully revised; second run; seven-day return usage.
- **Guardrail:** Attracting developers who expect Agenta to replace their coding environment.
- **Decision rule:** Advance if users repeatedly operate bounded workflows in Agenta. Narrow or reject if they only inspect the platform and return entirely to code.
- **Main risk:** Reliability is still rough, so “operate” must not imply production readiness that has not been demonstrated.

### Current prioritization

1. **Technical founders:** strongest founder/product-story fit.
2. **AI and automation builders:** strongest fit with the reachable mailing-list and GitHub audience.
3. **Self-hosting/local-AI builders:** strongest distribution and trust wedge, with the highest risk of architecture-led curiosity without workflow retention.

### Shared experiment architecture

- Give each hypothesis its own landing page, workflow demonstration, acquisition tags, and PostHog cohort.
- Keep GitHub as the open-source technical-evaluation destination, while making the landing page explain the user outcome.
- Compare qualified GitHub/install behavior, meaningful activation, repeated runs, and community contribution separately.
- Use small audience batches and controlled paid-message tests before exposing the full existing audience.
- Do not select a winner from CTR alone; require downstream activation and return behavior.
