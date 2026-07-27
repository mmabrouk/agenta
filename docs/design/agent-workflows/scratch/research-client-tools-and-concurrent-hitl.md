# Research: client-tool delivery per harness/sandbox, and concurrent HITL

Read-only research against CURRENT `gitbutler/workspace` HEAD (post sandbox_agent decomposition,
`services/runner/src/engines/sandbox_agent/`). Every claim cited `file:line`. Two sections, one
per question, each closing with hard constraints any design must respect.

---

## QUESTION 1 — How client tools reach each harness/sandbox combination, and what breaks for Claude on Daytona

### Where tool specs come from (run-plan.ts)

`buildRunPlan` reads `request.customTools` into `toolSpecs`
(`run-plan.ts:363`) and computes the executable subset
`executableToolSpecsForRun = executableToolSpecs(toolSpecs)` (`run-plan.ts:364`). `executableToolSpecs`
filters OUT client-kind tools; the plan carries both lists separately (`run-plan.ts:149-150`,
returned at `run-plan.ts:544-545`). The relay is armed whenever any tool exists:
`useToolRelay = toolSpecs.length > 0` (`run-plan.ts:550`).

The shared pause seam for a `client` tool is `buildClientToolRelay` (`client-tools.ts:205-268`).
Its docstring names the two delivery channels that both consume it: Pi via the file relay
(`tools/relay.ts` → `startToolRelay`), Claude-local via the loopback MCP server
(`tools/tool-mcp-http.ts`) (`client-tools.ts:6-15`).

### The four combinations

**Pi local — file relay.** Pi delivers tools through its bundled EXTENSION, never over ACP MCP;
`buildSessionMcpServers` returns `[]` for Pi (`mcp.ts:305-313`). Tools flow through the runner
file relay (`run-turn.ts:398-412`, host = `localRelayHost`). A `client` tool parks inside the relay:
`executeRelayedTool` sees `spec.kind === "client"`, calls `clientToolRelay.onClientTool`, and on a
`pendingApproval` decision calls `clientToolRelay.onPause` and returns `PAUSED`
(`relay.ts:326-345`). So **client tools park on Pi-local.**

**Pi Daytona — file relay over the sandbox FS.** Same extension + file relay, but the relay host is
`sandboxRelayHost` (`run-turn.ts:400-403`), which polls the sandbox filesystem. The client-tool park
path is the same relay code (`relay.ts:326-345`). So **client tools park on Pi-Daytona too** (the
relay round-trips the sandbox FS; the ClientToolRelay lives in runner memory and the relay loop that
services request files runs on the runner).

**Claude local — loopback HTTP MCP.** Tools ride the internal loopback (`127.0.0.1`) HTTP MCP server
`buildToolMcpServers` (`mcp.ts:327-332`). The `clientToolRelay` is passed into
`buildSessionMcpServers` ONLY on the non-Daytona branch (`mcp.ts:251-252`, consumed at
`mcp.ts:327-332`), so the `tools/call` handler in `tool-mcp-http.ts` pauses for a client tool through
the same seam. So **client tools round-trip on Claude-local.**

**Claude Daytona — in-sandbox stdio MCP shim (commit afb00d4218).** The loopback URL is unreachable
from inside the sandbox (its `127.0.0.1` is the sandbox's), so the channel becomes the uploaded stdio
shim: `buildInternalToolMcpEntry` advertises a TYPELESS `{name:"agenta-tools", command:"node",
args:[bundle], env}` ACP entry (`mcp.ts:109-134`) that the Claude ACP adapter maps to a Claude SDK
stdio MCP server. It is built ONLY when `internalToolMcp && executableToolSpecs(toolSpecs).length > 0`
(`mcp.ts:333-337`). The shim (`tool-mcp-stdio.ts`) serves `tools/list` from the uploaded public-specs
file (`tool-mcp-stdio.ts:193-208`) and on `tools/call` writes a relay request file via `relayToolCall`
and BLOCKS awaiting the runner's response (`tool-mcp-stdio.ts:210-254`, the awaited call at
`tool-mcp-stdio.ts:226-232`).

### The precise mechanical reason client tools cannot round-trip on Claude-Daytona

Three independent mechanical blocks, all in current code:

1. **Client specs are never advertised to the model.** The uploaded specs file is built from the
   executable subset only, and the shim entry is constructed only when
   `executableToolSpecs(toolSpecs).length > 0` (`mcp.ts:333`). `run-plan.ts` omits client tools from
   the Daytona shim path deliberately (`run-plan.ts:425-442`, comment at `run-plan.ts:73-90`). So
   Claude-in-Daytona never sees a client tool in `tools/list` and cannot call one.

2. **The shim's `tools/call` is a synchronous request/response with no "park" outcome.** It maps a
   call to exactly one of: a text result, or an `isError` result (`tool-mcp-stdio.ts:233-253`). There
   is no frame for "pause this turn, resume next turn." It awaits a `.res.json` bounded only by the
   per-tool `spec.timeoutMs` (`tool-mcp-stdio.ts:230`).

3. **The only shim↔runner channel is the FILE RELAY**, a one-request/one-response file exchange
   (`<uuid>.req.json` → `<uuid>.res.json`) whose env is just `relayDir` + `specsPath`
   (`mcp.ts:113-116`, `tool-mcp-stdio.ts:52,226`). The shim holds NO `ClientToolRelay` object — that
   lives in runner memory (`client-tools.ts`) and is passed to the MCP server only on the local branch
   (`mcp.ts:251-252`). A client-tool round-trip needs a runner→frontend hop mid-call; the file relay's
   req/res shape cannot express it.

**Could the existing runner↔sandbox control connection carry a tool-call-back?** The sandbox-agent
daemon / ACP link (built via `createAcpFetch`, `daemon.ts`) carries the ACP protocol only: session
prompt, `session/update` events, and the permission reverse-RPC
(`onPermissionRequest`/`respondPermission`). That reverse-RPC IS the suspension point for
HARNESS-raised permission gates — this is exactly how Claude approvals park
(`acp-interactions.ts:147-182`, `pauseUserApproval`). But a client tool is NOT an ACP permission gate;
it is an MCP `tools/call` the shim resolves over the file relay, out-of-band from the ACP link. So the
control connection does not today carry an MCP client-tool callback. (Even the runner-side relay loop's
own client-park path, `relay.ts:341-344`, is moot on Daytona because the shim never advertises a client
tool, so no request file is ever written for one.)

### Where the park/resume machinery hooks in

- `ParkedApproval` (`runtime-contracts.ts:113-128`) is recorded ONLY for the two ACP permission gate
  types (`ParkedApprovalGateType = "claude-acp-permission" | "pi-acp-permission"`,
  `acp-interactions.ts:24-26`). It is explicitly NOT recorded for a client-tool MCP pause
  (`runtime-contracts.ts:105-111`; `acp-interactions.ts:63-67`).
- `ResumeApprovalInput` (`runtime-contracts.ts:131-139`) answers a parked gate live via
  `session.respondPermission` (`run-turn.ts:427-457`) — approvals only.
- Client-tool resume is the COLD-REPLAY path exclusively: the browser output is read back from replayed
  history by `extractClientToolOutputs` (`responder.ts:391-409`), and the transcript builder emits a
  `client` resume frame (`resumeKindFor` "client", `transcript.ts:195,217-246`; `CLIENT_RESUME_CLOSING`,
  `transcript.ts:190-193`). This cold path exists for Claude on Daytona too — but is never reached for a
  client tool, because the shim refuses/omits client tools up front (block #1 above).

### What #5366 does today

Mapped to `run-plan.ts:425-442`:

- **Refuse when the tool set is ONLY client tools:** on `!isPi && isRemoteSandbox && isDaytona &&
  toolSpecs.length > 0`, if `executableToolSpecsForRun.length === 0` it returns
  `DAYTONA_CLIENT_ONLY_TOOLS_UNSUPPORTED_MESSAGE` (`run-plan.ts:439-441`; message at
  `run-plan.ts:84-90`). Without this the run would proceed, advertise an empty tool set, and return
  `ok:true` (the "F1 zero-tools drop").
- **Drop client tools silently in a MIXED set:** when some executable tools remain, the run proceeds;
  the shim's uploaded specs file is `executableToolSpecs(toolSpecs)` only (`mcp.ts:333,344`), so client
  tools are simply not advertised (dropped) while executable ones are delivered. The log at
  `mcp.ts:341-348` states honestly "N gateway tool(s) advertised", not the client count.
- A non-Daytona remote provider fails closed entirely: `REMOTE_TOOLS_UNSUPPORTED_MESSAGE`
  (`run-plan.ts:433-434`, message `run-plan.ts:66-71`).

### Hard constraints (Section 1)

- Any Claude-Daytona client-tool design must add a NEW park/resume channel; the file relay's
  one-req/one-res shape and the shim's blocking synchronous `tools/call` cannot express a pause
  (`tool-mcp-stdio.ts:210-254`).
- Client tools are stripped before the shim's specs upload (`mcp.ts:333`, `run-plan.ts:425-442`); a
  design must first make the shim ADVERTISE client tools before the model can ever call one.
- Parked-approval machinery (`ParkedApproval`/`ResumeApprovalInput`/live `respondPermission`) is built
  for ACP permission gates only and deliberately excludes client-tool pauses
  (`runtime-contracts.ts:105-128`); client-tool resume today is cold-replay only (`responder.ts:391-409`,
  `transcript.ts:190-246`).
- The ACP control link carries only harness permission reverse-RPC as a suspension point
  (`acp-interactions.ts:147-182`); it is not a general tool-callback channel.
- The `agenta-tools` server name is reserved and coupled to rendered permission rules
  (`mcp.ts:71-76`, gate at `run-plan.ts:417-423`) — a new channel must not collide with it.

---

## QUESTION 2 — Two concurrent human-in-the-loop requests

### Can a turn park with more than one pending interaction?

Two distinct answers by interaction kind:

- **APPROVAL gates: NO — one per turn.** Every approval pause is gated by a one-shot latch,
  `PendingApprovalLatch.tryAcquire()` (`permission-plan.ts:173-185`: first call `true`, all later calls
  `false`). `pauseUserApproval` calls `if (!latch.tryAcquire()) return;` BEFORE emitting the card
  (`acp-interactions.ts:164`). A second concurrent approval gate: its `onUserApprovalGate` still fires
  and increments the counter (`acp-interactions.ts:154-163`, counter in `run-turn.ts:350-368`), but the
  latch blocks the emit — no second `interaction_request` goes out. The sibling gated tool call is then
  force-settled as `TOOL_NOT_EXECUTED_PAUSED` (`run-turn.ts:230-236` and the post-drain re-sweep at
  `run-turn.ts:482-488`; the pause controller's destroy callback also settles announced-but-unresolved
  siblings, `pause.ts:1-8`).

- **CLIENT tools (relay/MCP seam): YES — multiple park in one turn.** `buildClientToolRelay` does NOT
  consult the latch: "Client tools do NOT gate on it — each pending client tool parks its own widget"
  (`client-tools.ts:178-181`), and it emits a `client_tool` interaction for EACH pending call
  (`client-tools.ts:245-264`). The turn still ends exactly once — `pause.pause()` via `onPause` is
  idempotent (`client-tools.ts:248-250,266`; idempotence at `pause.ts:27-42`). (Note: the SEPARATE
  ACP-gate client-tool path `pauseClientTool` DOES take the latch, `acp-interactions.ts:190`; but the
  primary Pi-relay / Claude-local-MCP client delivery goes through the un-latched `client-tools.ts`
  seam.)

### What the wire emits

One `tool-approval-request` chunk per `user_approval` `interaction_request` event
(`stream.py:680-684`, in `_interaction_parts`). Because the latch caps approvals at one emit per turn,
the wire carries at most one approval-request per turn. Client-tool events map one part each
(`stream.py:689+`), so several client widgets can stream in one turn. Interaction requests are
projected sequentially as they arrive; there is no batched multi-approval frame.

### What extractApprovalDecisions accepts

`extractApprovalDecisions` returns `Map<string, unknown[]>` — a LIST of decisions per cold-replay key
(`responder.ts:358-373`, appended at `responder.ts:368-370`). So a single inbound resume request CAN
carry MANY decisions (across different tool-call keys, plus a FIFO list per identical-args key).
`ConversationDecisions.take` consumes one per key FIFO (`responder.ts:256-263`, `queue.shift()` at
`responder.ts:263`). But because only one gate pauses per turn (the latch), those stored decisions are
consumed one gate at a time as gates RE-RAISE across successive cold-replay turns — the map holds many;
each turn answers one.

### FE resume predicate with a second unanswered request

`agentShouldResumeAfterApproval` (`agentApprovalResume.ts:131-165`): it finds the LAST freshly-resolved
parked interaction (`agentApprovalResume.ts:145-149`; comment "a second approval later in the turn is
what should drive the (next) resume", `:143-144`) and then requires
`allSettled = toolParts.every(isSettledToolPart)` (`agentApprovalResume.ts:163-164`). So if a SECOND
interaction is still pending (an unsettled `approval-requested`, or an `input-available` client widget),
`allSettled` is false and the run does NOT auto-resume — it waits until the human answers every pending
interaction. Separately, `isPendingClientToolInteraction` (`agentApprovalResume.ts:93-100`) holds the
message queue while any client widget is still live, so a queued user message cannot inject before the
widget settles.

### What the SDK ingress does with a second request

The ingress does not block; it serializes. Each `approval-responded` tool part converts to a
`tool-result` block keyed by `toolCallId` (`messages.py:177-207`, `_approval_response_blocks`
`messages.py:224-227`; decision read at `messages.py:210-222`). Multiple responded parts → multiple
tool-result blocks → all picked up by `extractApprovalDecisions`. So concurrency is resolved by
folding each answer into message history, not by an in-flight second-request handler.

### Where the "second one gets queued/blocked" behavior lives

- **Runner (approvals):** the latch (`permission-plan.ts:173-185`, `acp-interactions.ts:164`) — one card
  per turn; siblings force-settled (`run-turn.ts:230-236,482-488`).
- **Runner keep-alive:** a multi-gate turn cannot live-park. `env.parkedApproval` records only the FIRST
  gate (`run-turn.ts:355-366`), and the dispatch refuses to park when `approvalGateCount > 1`:
  `multi-gate-no-park` (`server.ts:428-431`), falling back to the cold decision-map path. A non-parkable
  (client-tool) pause also never parks: `non-parkable-gate-no-park` (`server.ts:424-426`).
- **FE:** all-settled gate (`agentApprovalResume.ts:163`) plus the pending-client-tool queue gate
  (`agentApprovalResume.ts:93-100`).

### Pi relay S5.2 gap — can relay tools park at all?

Documented open issue `open-issues.md:132-169`: the runner relay loop is a synchronous fire-and-forget
poll with NO turn-boundary model, so a resolved code/gateway tool with an `ask` permission cannot emit
an `interaction_request`, end the turn, and resume the same call later. The suspension mechanism the
harness path relies on (the ACP permission request itself blocking on `respondPermission`) does not
exist for a relay execution — "the relay just executes and returns a string inline"
(`open-issues.md:155-158`). Two current-code qualifiers:

- **Client tools are the exception** — they DO have an explicit park path in the relay
  (`relay.ts:326-345`), added after the S5.2 note; so "relay tools cannot park" is specifically about
  executable code/gateway `ask` tools, not client tools.
- **Pi APPROVAL asks are routed around the relay:** a Pi gate rides the extension's `ctx.ui.confirm`
  onto the ACP permission plane and parks via `pauseUserApproval` with gate type `pi-acp-permission`
  (`acp-interactions.ts:289-341`, `relay.ts:314-317`). So Pi custom-tool/builtin approvals park through
  the ACP machinery; the narrow residual S5.2 is a relay-executed `ask` that would need to park FROM the
  relay loop.

### Plainly: what is possible / serialized / broken today

- **Possible now:** exactly one approval card per turn (any harness); multiple client-tool widgets
  parked together in one turn (`client-tools.ts:245-264`); many stored decisions carried in one cold
  resume, consumed one gate per turn (`responder.ts:358-373`).
- **Serialized:** true-concurrent approval gates. Only the first emits; the rest force-settle
  (`TOOL_NOT_EXECUTED_PAUSED`) and the model re-requests them one at a time across cold-replay turns.
  Keep-alive explicitly declines to live-park a multi-gate turn (`server.ts:428-431`).
- **Broken / unsupported:** a live (keep-alive) resume of more than one pending approval — no data
  structure exists for it (`ParkedApproval` is singular, recorded only for gate #1,
  `run-turn.ts:355-366`); and a relay-executed `ask` code/gateway tool cannot park at all (S5.2,
  `open-issues.md:132-169`).

### Hard constraints (Section 2)

- The approval pause is single-shot per turn by construction (`permission-plan.ts:173-185`,
  `acp-interactions.ts:164`); any multi-approval design must replace or relax the latch AND add
  force-settle handling for the extra gates (`run-turn.ts:230-236`).
- `ParkedApproval` is singular and records only the first gate (`runtime-contracts.ts:113-128`,
  `run-turn.ts:355-366`); live multi-gate resume needs a plural park record and a multi-answer resume
  input (today `ResumeApprovalInput` answers one, `run-turn.ts:427-457`).
- The keep-alive dispatch hard-refuses `approvalGateCount > 1` (`server.ts:428-431`); relaxing it
  requires answering multiple `permissionId`s on the live session in one resume.
- The FE resume predicate requires ALL parked interactions settled before resuming
  (`agentApprovalResume.ts:163`) — a concurrency design must keep this or the queue-gate invariant
  (`agentApprovalResume.ts:93-100`) holds forever.
- The decision store already supports many decisions per resume (`responder.ts:358-373`) and client
  tools already park in parallel (`client-tools.ts:245-264`); the cold-replay path is the existing
  precedent for multiplexing HITL — a live design must not regress it.
- Relay-executed `ask` tools have no park path (S5.2, `open-issues.md:132-169`); any design that routes
  executable-tool approvals through the relay must first give the relay a turn-boundary model.
