# Channels: architecture

Date: 2026-07-20. Status: second-pass synthesis, for review. Inputs: the decision
record (`decisions.md`), the codebase audit (`raw/agenta-primitives.md`), the
open-source gateway study (`raw/oss-gateways.md`), the integration-contract study
(`raw/integration-sdks.md`), the enterprise security research
(`raw/enterprise-posture.md`), and a divergent brainstorm from GPT-5.6
(`raw/gpt56-brainstorm.md`). Where sources disagreed, this document says which way
we go and why.

**How to read this.** Section 1 states the design in one paragraph. Section 2
introduces the seven new nouns and maps each onto what already exists in the
codebase. Sections 3 and 4 walk one message inbound and one reply outbound, which
is where most of the design lives. Section 5 covers identity and the security
invariants. Section 6 is the extension contract for channels we do not build
ourselves. Section 7 is the build plan with effort estimates. Section 8 holds the
open questions.

## 1. The design in one paragraph

Channels are a projection of our existing session subsystem onto messaging
surfaces, not a new conversation system. One new service, the **channel
gateway**, sits between messaging platforms and the platform API. It terminates
platform connections (Slack Socket Mode, Telegram long polling, bridge
connections from third parties), normalizes what arrives into a small typed
activity vocabulary, resolves which agent is being addressed and which session
the conversation maps to, and invokes the agent through the same `/invoke`
contract every other surface uses. On the way back it consumes the runtime's
event stream, renders replies and approval buttons within each platform's
declared capabilities, and posts them with delivery receipts. Everything
stateful it needs (sessions, transcripts, approvals, credentials, identity
links, event delivery) already exists in the platform; the gateway adds exactly
one new domain (channel bindings and their conversation links) plus a wire
contract that lets a self-hosted user run their own bridge for a platform we
cannot support ourselves.

Two properties fall out of this shape and are worth naming. First, the web app
and a Slack thread are peers: both are surfaces reading and writing the same
session, so cross-surface continuity is a property of the session subsystem, not
a channel feature. Second, the agent runtime never touches platform credentials
or destinations: it emits events against a session, and the gateway alone
resolves where and how those become Slack messages. That boundary is the single
highest-leverage security control in the design (see 5.4).

## 2. The nouns, and what each reuses

The brainstorm proposed three decompositions and ranked an explicit
conversational vocabulary first; we adopt that ranking, with one deliberate
deviation: where GPT-5.6 modeled the platform-app installation as a brand-new
`ChannelApp` store, we keep the credentialed installation in the existing
connections subsystem and let the new channel domain reference it. The audit's
hard rule ("what must NOT happen is a second connection store or a second
reference-resolution scheme") outweighs the aesthetic of a self-contained
domain, and the objection GPT-5.6 raised against reusing connections (conflating
a data-provider connection with a bot installation) is answered by the provider
kind: `gateway_connections` already enumerates `composio | agenta`, and a native
`slack` provider adapter is the anticipated, unimplemented slot.

| Noun | What it is | Where it lives |
|---|---|---|
| Channel app | One bot identity installed on one platform tenant: a Slack app in a workspace, a Telegram bot, an email address. Decision D3: one app is one agent's identity. | A `gateway_connections` row under a new native provider (`slack`, `telegram`, `bridge:<name>`); bot token and signing secret are vault secrets referenced from it. New code: the native provider adapter filling the existing `ConnectionsService` port. |
| Channel binding | The active relationship "this channel app drives this agent", plus conversation policy: thread-to-session mapping, session lifetime, DM policy, group invocation mode. | New table in a new `channels` domain, a sibling of `TriggerSubscription`: same `Reference` latest-binding to the agent, same connection row underneath, same queue-and-dispatcher shape. Triggers fire one run; bindings converse in sessions; they stay siblings sharing parts, per both the audit and the brainstorm. |
| Scope grant | Default-deny exposure: the explicit list of workspaces/channels/DM contexts where the binding may answer, each with an invocation mode (mention, command, always). | Child rows of the binding. Not a public API noun in v1, but a table from day one because retrofitting default-deny is a migration and an enterprise-review finding. |
| Conversation link | The durable correlation "this external conversation is this session", keyed `(binding, native conversation key, generation)`. `generation` exists because session lifetime is config (D5): the same thread may map to session 1, then session 2 after expiry, without losing history. | New table in the channels domain. The native key is built by ONE core function from a structured locator the adapter supplies; adapters never mint session keys (the unanimous lesson of every gateway studied). Multiple links may point at one session, which is how a session later gains a second surface. |
| Identity link | "Slack user U in workspace T is Agenta user X." Required by D2 (the agent acts with the invoking user's permissions). | The existing `user_identities` table, method family `channel:slack:{team_id}`, subject = the platform's immutable user id. No schema change. Linking is deterministic code, never inference, and never by display name; email matching at most bootstraps a suggestion that the user confirms via a magic-link DM. |
| Inbox event | The durable, idempotent receipt of one inbound platform event, unique on `(channel app, external event id)`. Platforms redeliver aggressively (Slack retries at 3 seconds); dedup at the door makes everything downstream exactly-once in effect. | New table plus a receiver cloned from the implemented Composio trigger receiver: public-path allowlist, signature verify with replay protection, 202-accept, queue. Slack's `v0:{timestamp}:{body}` HMAC is structurally identical to what that receiver already verifies. |
| Delivery | One outbound projection (a reply, a status update, an approval card) with an idempotency key, attempts, and the platform's message id as receipt. Receipts are what make edits, threaded replies, and approval-card updates possible. | New outbox table in the channels domain, driven by the existing webhook delivery machinery's patterns (signing, retry, delivery log). We adopt the brainstorm's correction here: the reserved `delivered_webhook` boolean on interactions is too weak; an interaction relates to deliveries, since one approval may render on several surfaces and be re-rendered after updates. |

One noun from the brainstorm we adopt as a contract rather than a table: the
**delegation context**. Every turn a channel starts carries `subject` (the human
whose authority is exercised), `actor` (the agent), and `audience` (personal DM
or shared conversation). In the MVP these ride the existing credential and
session metadata; section 5.2 explains the migration path to real delegated
tokens. Recording all three from day one is the difference between "add a
column" and "re-audit every log statement" when enterprise review arrives.

## 3. Inbound: one Slack message, end to end

A user writes `@deploybot please ship v2` in a thread.

1. **Receive.** Slack delivers the event over Socket Mode (an outbound WebSocket
   the gateway holds open; no inbound port, which is the enterprise-default
   topology) or, for cloud installs, a signed webhook. The gateway verifies
   authenticity, writes the inbox event (deduping retries), and acks within
   Slack's 3-second window. Nothing else happens on the hot path; every
   platform SDK studied enforces some version of ack-fast-work-later, and
   Linear turns it into a formal 10-second SLO.
2. **Route.** The inbox event's app and team resolve to the channel app row,
   then to its binding and the scope grants. No grant covering this channel
   means no reply (default-deny). The binding's invocation mode decides whether
   a bare thread reply counts (inside a thread the agent owns, it does; outside,
   only a mention does).
3. **Identify.** The sender's Slack id looks up the identity link. Linked:
   the turn will run as that Agenta user (D2), checked against the live account
   state at this moment, not at link time, so a deactivated employee loses
   channel access the second SCIM deactivates them. Unlinked: the gateway
   replies with a link invitation for consequential requests; policy for
   read-only requests is a binding setting.
4. **Correlate.** The structured locator (team, channel, thread timestamp)
   resolves get-or-create against conversation links, minting a session on
   first contact. Get-or-create on external keys is also what makes redelivered
   webhooks harmless.
5. **Schedule.** The message enters the session's mailbox. The brainstorm
   called the mailbox the primitive everyone forgets, and our audit agrees with
   the diagnosis while softening the remedy: the stream command plane already
   serializes turns (`send` returns 409 `SessionTurnInUse`; `steer`
   interrupts), so what is missing is only a per-conversation FIFO in front of
   it. That queue lives once, in the gateway core, never per adapter. MVP
   policy is serialize; coalesce and interrupt become binding config later.
6. **Invoke.** The gateway calls `/invoke` with the session id, the agent
   reference from the binding, and the message, exactly as the trigger
   dispatcher already does server-side. Speaker attribution rides the message
   (name-prefixed in text until the `sender` field lands; see 7). Batch mode
   (one JSON response) is the natural default for messaging; token streaming is
   pointless where replies are posted as whole messages.

The prerequisite underneath step 6 is server-side history hydration: `/invoke`
with a known session and no shipped history must rebuild model context from the
stored records. The audit confirms this is the single generalization that makes
continuity real for every surface at once (the replay logic exists today only as
frontend TypeScript), and the brainstorm independently named it "a prerequisite,
not a channel feature". It is the first thing we build (7).

## 4. Outbound: replies, progress, and approvals

The runtime already emits a protocol-neutral event stream (`AgentEvent`: text,
reasoning, tool calls, interaction requests, files, errors, done). The gateway
consumes a curated projection of it, not the raw browser-oriented SSE: message,
status, tool activity, interaction, result, error. Two deliberate exclusions,
both on GPT-5.6's advice and both accepted: chain-of-thought never leaves the
platform as a channel activity (a status line is portable and safe; reasoning
is neither), and channels never receive an open-ended payload bag they can
start depending on.

Rendering is capability-driven. Every channel app declares, as data, what its
surface supports: threads, edits, buttons and how many, message length and its
unit, markdown dialect, files. The gateway plans behavior from the declaration
(progress via message edits where edits exist, message-per-update where not;
buttons where buttons exist, "reply 1 to approve" where not), and contract
tests hold adapters to their declarations, because the silent no-op is the
canonical adapter failure mode. Where the declaration comes from is the same
for first-party and third-party channels (see 6).

Approvals are the strongest case for this whole feature, because the runtime
half already exists end to end: a gated tool call creates a durable interaction
row, the unanswered approval parks the turn (an approval answered hours later is
a normal resume, not a held-open connection), and
`POST /sessions/interactions/{id}/respond` resumes the run server-side against
the pinned revision. No browser hop, which is precisely the top complaint about
n8n's and Zapier's channel approvals. What we add:

1. **Push.** Interaction-created and turn-completed events join the platform
   event bus (`EventType`/`WebhookEventType`), so the gateway is notified
   rather than polling. The delivery machinery for this exists; only the event
   types are missing.
2. **Render.** The gateway posts the approval as an interaction activity built
   from the structured tool-call data, never from model-composed text (a
   prompt-injected agent must not be able to compose its own approval card).
3. **Authorize the click.** A button click is presentation, not authorization.
   On click the gateway verifies the platform payload, resolves the clicker
   through identity links, and re-evaluates who-may-approve policy before
   calling respond. Approver policy is a binding-level setting distinct from
   who-may-trigger; today the runtime accepts any project member with run
   rights, and the policy hook slots into the respond path. Possession of a
   callback value is never authorization.
4. **Update.** The delivery receipt lets the gateway rewrite the card after
   resolution ("Approved by Alice, 14:02"), including on other surfaces
   showing the same interaction.

Because the same interaction is addressable from anywhere, the same approval
can be answered from the web app or from Slack, whichever comes first; the CAS
flip on the row guarantees exactly-once resolution. That, plus approver roles,
is our real differentiator now that in-channel approval buttons themselves have
shipped elsewhere (see `decisions.md` on Gumloop).

## 5. Identity, authorization, and the enterprise invariants

### 5.1 Whose permissions

D2 fixes the model: the agent acts with the invoking user's permissions. The
verification pass showed Anthropic deliberately chose the opposite for shared
channels (admin-provisioned service accounts, "what Claude can do never changes
based on who asked") for predictability's sake. We are choosing per-user
authority anyway, for least privilege and per-person audit, and we buy back the
predictability risk with three compensations: the group-safety rule stays in
the architecture (5.3), the subject/actor/audience triple makes every action
attributable, and the identity check runs per invocation. The door to
service-account and agent-own-identity modes stays open as an
`authorization_mode` on the binding, which is also where Claude-Tag-style
provisioning would land if a customer wants it.

### 5.2 Credentials, in two steps

MVP: the gateway holds one project credential per linked user (the existing
per-user API key, today's only delegation primitive) and invokes as the linked
user. Workable, auditable, and ugly at scale. Step two: a delegated-token
endpoint that formalizes the internal short-lived Secret JWT the middleware
already mints, so the gateway authenticates as itself once and exchanges
`(identity link, project)` for a scoped, expiring token per turn. That same
minting path is where a future non-human principal (the agent as its own
identity) fits, which is why building it as token exchange rather than more API
keys is the door D2 requires us to keep open. The audit sizes this at 2+ weeks
and confirms the MVP can ship before it.

### 5.3 The group-safety rule, accommodated not implemented

The rule (a group-visible reply must never use resources only one member may
access) is out of MVP scope by decision D1, and currently costs nothing:
member-private resources do not exist yet, since secrets and connections are
strictly project-scoped, and every credential and tool enters a run through one
SDK resolution chokepoint. The architecture work is to not break that property:
resolution stays behind the single chokepoint, sessions carry a
`context_kind: personal | shared` flag from day one (the conversation link
already knows), and tool outputs gain provenance the moment per-user resources
appear. MVP behavior, when that day comes, is the conservative one Copilot
Studio ships: refuse member-private sources in shared conversations outright.

### 5.4 The invariants a CISO will hold us to

From the enterprise research, restated as design rules this architecture
satisfies by construction, so they cannot regress silently:

1. **Egress-only by default.** Socket Mode and long polling need no inbound
   port; the webhook path exists for cloud and for Teams, which structurally
   requires public HTTPS and gets documented honestly rather than promised
   away.
2. **Bring your own app.** Self-hosters create their own platform apps from
   manifests we ship; their tokens live in their vault, encrypted with their
   key, rotatable, and never touch us. This is a security headline, not a
   setup burden.
3. **Platform credentials never enter the agent's execution context.** The
   runtime emits activities against a session; only the gateway resolves
   credentials and destinations, and replies are pinned to the originating
   conversation. Even a fully prompt-injected agent cannot exfiltrate to a
   channel of its choosing or read a token it was never given.
4. **Files cross a boundary.** Platform attachments are fetched by the
   gateway, stored via the existing session mounts, and enter the sandbox as
   scoped asset references, never as platform download URLs with embedded
   credentials.
5. **Audit carries subject, actor, and audience.** "Agent X, acting for Alice,
   in shared conversation Z, called deploy, approved by Bob" is expressible in
   structured audit events from the first release, exportable to a SIEM, with
   retention configured separately from message content, which is minimized
   (the mention thread, not channel history) and deletable.
6. **Revocation has defined semantics.** Unlinking an identity, disabling an
   app, or deactivating a user each states what happens to future invocations,
   pending approvals, queued deliveries, and history (tombstones where audit
   forbids hard deletion). Designed now because deletion semantics are
   unretrofittable.

## 6. Extensibility: the bridge contract

The requirement: a self-hosted user must be able to add a channel we cannot
build or even test (WeCom needs a verified Chinese enterprise tenant), without
forking core and without waiting for our releases. Both research tracks
converged from opposite directions on the same answer, and the Tencent WeCom
team maintaining an OpenClaw channel plugin with zero core changes is the
existence proof: the extension mechanism is a **versioned wire contract**, and
in-process plugins are explicitly rejected as the third-party story because
in-process code is RCE-equivalent trust (OpenClaw's malicious-registry incident
is the cautionary tale, and matterbridge's death by adapter bit-rot is the
maintenance one).

The contract, synthesized from Chatwoot's API channel, Direct Line, LangBot's
http_bot, Hermes's relay adapter, and the brainstorm:

- **One generic "remote channel" provider in core.** A bridge is a small
  stateless service the user runs, in any language, that speaks the platform's
  native API on one side and this contract on the other. To Agenta it IS a
  channel app: same connection row, same binding, same capability declaration,
  same rendering pipeline. Nothing downstream knows it is third-party.
- **Bridge dials out.** Default transport is a long-lived outbound WebSocket
  from the bridge to the gateway (matching the runner's dial-out story and the
  egress-only invariant); an HTTP profile (signed webhooks both ways) exists
  for bridges that prefer it. Same envelopes on both.
- **Capability hello.** On connect, the bridge presents protocol versions and
  its frozen capability descriptor. Hostile or absurd values are normalized at
  the trust boundary; trust flags are stamped host-side, never read from the
  wire. The bridge credential authorizes transport only; human authorization
  always goes through identity links per invocation, and one bridge cannot
  speak for another installation.
- **Envelope and evolution.** CloudEvents 1.0 as the outer envelope with W3C
  trace context, our activity schema inside, version in the type name, additive
  evolution under a written must-ignore rule, and named versioned extensions
  instead of an open `channelData` bag. The event vocabulary is the Activity
  checklist: message received/updated/deleted, reaction, membership change,
  interaction response, plus delivery commands and receipts the other way.
- **Idempotency and ordering.** Stable external event ids inbound, idempotency
  keys outbound, at-least-once delivery, ordering guaranteed per conversation
  only, core-owned resume cursors so a bridge can be replaced mid-stream.
- **First-party channels eat this contract.** Slack and Telegram ship as
  bundled bridges speaking the same protocol (in-process transport, same
  envelopes), which is the Bot Framework connector model and the only known way
  to keep a third-party contract honest. If our own Slack support cannot be
  built on the contract, the contract is wrong, and we find out first.

Distribution governance (a directory of community bridges, a verified tier) is
explicitly deferred: the wire contract makes distribution anyone's problem, and
the one rule worth pre-committing to is that community code is never
auto-installed without curation.

## 7. Build plan: reuse first

What the feature actually requires beyond configuration, ordered so that each
step is independently shippable. Estimates from the codebase audit.

1. **Session and interaction events on the event bus** (days). Unblocks push
   delivery of approvals and turn completions to any surface, including the
   web app's own notifications.
2. **`channel:*` identity methods and the linking flow** (days plus UX). The
   `user_identities` table needs no schema change; the flow is a bot DM with a
   magic link.
3. **Slack receiver** (days). Clone of the Composio ingress: allowlist, HMAC,
   202, queue. Socket Mode client lives in the gateway worker.
4. **Channels domain: binding, scope grants, conversation links, inbox,
   deliveries** (about a week with CRUD and UI). The new tables of section 2.
5. **Native Slack connection provider** (about a week including token
   rotation). Fills the existing `ConnectionsService` port; app-manifest flow
   for self-hosters, OAuth install for cloud.
6. **Server-side history hydration** (1 to 2 weeks). The continuity
   generalization; port the records-to-messages replay server-side and make
   `/invoke` hydrate when history is absent. Benefits every surface, not just
   channels.
7. **Speaker attribution: `sender` on messages and records** (1 to 2 weeks
   across wire, records, replay, UI). Until it lands, the gateway prefixes
   names in text, which is exactly what every surveyed vendor does.
8. **Delegated tokens** (2+ weeks, after MVP). Replaces per-user API keys with
   token exchange; opens the service-account door.
9. **The bridge contract** (spec first, then the generic remote-channel
   provider). Slack ships first as a bundled bridge on the internal form of
   the contract; the external WebSocket/HTTP profile follows once the schema
   has survived first contact with Telegram, the second channel that keeps the
   abstraction honest.

The MVP line: steps 1 through 6 give Slack end to end (mention opens a session,
team converses in the thread, session continues in the web app, approvals
resolve in-channel) on per-user API keys with name-prefixed attribution.
Everything after hardens and generalizes.

## 8. Open questions

1. **Where the gateway runs.** Socket Mode and long polling need an always-on
   process, so the gateway is a worker service in the compose stack, not
   request-scoped API code. Recommended: a new small service next to the
   existing worker entrypoints, sharing the API's task-queue infrastructure.
   The alternative, folding it into the existing worker process, saves a
   container but couples channel connectivity to worker deploys.
2. **Publish the bridge contract when?** Recommended: design the schema now,
   publish after Telegram proves it (two first-party channels before one
   external promise). Publishing with v1 would invite bridges against a
   contract that has never survived a second channel.
3. **MVP approval rendering.** Buttons on Slack from day one (it is the flagship
   moment of the feature), text fallback ("reply 1 to approve") as the generic
   path all channels get free. Cost is small since the interaction activity is
   already structured.
4. **DM sessions and billing/attribution.** DMs are allowed (D4) and
   mechanically identical (a conversation link with `context_kind: personal`).
   Whether DM usage should be attributed or limited differently from channel
   usage is product policy we have not discussed.
