# Engine quality roadmap: making the sandbox-agent engine well-written

**Date:** 2026-07-19
**Audience:** Mahmoud first (senior architect, new to TypeScript), then the implementer
agents who will do this work. Every TypeScript-specific word is defined in one plain sentence
the first time it appears.
**Status:** design and TODO list. Read-only assessment of the code as it stands on the lane
`plan/client-tools-daytona` (PR #5383). No code was changed to write this.

This document answers a specific complaint. Reviewing PR #5383, you looked at
`services/runner/src/engines/sandbox_agent/environment.ts` and said it is "not very well
written, probably not from that PR." You are right on both counts, and the reason is worth
stating plainly before we plan the fix, because the fix depends on understanding why the code
looks the way it does.

A note on one word used throughout. A *module* (or *file*) here is a single `.ts` file. When
this document says a file "does too many jobs," it means the file mixes several unrelated
concerns that a reader has to hold in their head at once.

---

## 1. Why the code looks the way it does

`environment.ts` is not the product of PR #5383. Almost all of it is code that existed months
ago, moved verbatim into a new file. Here is the history.

The runner is the Node service that drives a coding agent inside a sandbox. (Node is a runtime
for running JavaScript and TypeScript on a server.) Its core used to be one file,
`sandbox_agent.ts`, that had grown to about 2,477 lines and did everything: plan the run,
acquire the sandbox, mount storage, wire up tools, open the agent session, run each turn, and
tear it all down.

In July we ran a "desloppify" pass on that one file. The shipped work, PR #5369 (in release
v0.105.5), was deliberately **Phase 1 only: pure decomposition, moves and nothing else.** The
rule for that pass was strict and correct: change *where* code lives, never *what it does*.
Every test that passed before had to pass after, the network messages had to stay byte-for-byte
identical, and any bug found while moving code was written down and left in place, not fixed.
That is why the 2,477-line monolith became a 44-line facade (a *facade* is a small file whose
only job is to re-export names from sibling files, so the rest of the codebase has one stable
place to import from) plus about ten sibling files, one of which is `environment.ts`.

The value Phase 1 bought is real and should not be undersold: the giant file is gone, each
concern now has a named home, the eight extraction commits each stayed green so the history
bisects cleanly, and the whole thing survived the agent release gate (the wire-level end-to-end
QA harness) across both harnesses and both sandboxes. It also survived two later rebases (onto
v0.105.4 and JP's sessions work) without behavior drift. That is a solid foundation.

But Phase 1 was explicitly the *first* of three phases, and the other two were deferred with
written designs. The restructuring that would make each file actually *well-written* (grouping
files into subsystems (Phase 2) and drawing clean seams so the code stops asking "is this Pi? is
this Daytona?" at every step (Phase 3)) was never done. So `environment.ts` today holds the old
`acquireEnvironment` function body, roughly 1,100 lines, almost exactly as it was inside the
monolith. It was moved, not rewritten. Your instinct that it is "probably not from that PR" is
precisely correct: PR #5383 (client tools on Claude in Daytona) touched only a handful of lines
inside it; the shape you dislike is inherited.

This document takes the code as it is now and lays out how we make it good, in follow-ups that
each stand alone.

---

## 2. An honest per-file assessment of the engine as it stands

The engine lives in `services/runner/src/engines/sandbox_agent/`. It is about 40 files. Most are
small, single-purpose, and fine. The assessment below names only the files that carry debt; the
unnamed ones (for example `pause.ts`, `usage.ts`, `errors.ts`, `capabilities.ts`) are healthy.

### 2.1 `environment.ts` (1,077 lines): the one that reads badly, dissected

This file exports `acquireEnvironment`, the function that builds everything a session needs
before the first turn runs. The problem is not its length on its own. The problem is that one
function interleaves at least eight distinct jobs, threads a single mutable object through all of
them, and branches on the harness type and sandbox type at nearly every step. Here are the eight
jobs and roughly where each one lives in the function today:

| Job | What it does | Where it lives now |
|---|---|---|
| Preflight gating | Throw before any resource if a fail-closed condition holds (model config unwritable, permission extension missing) | lines 566–580 |
| Local mount wiring (pre-start) | Mount the durable working directory and agent directory on the host before the daemon spawns | lines 588–596 |
| Sandbox acquisition | Build the provider, reconnect a stored sandbox by id or create a fresh one | lines 597–665 |
| Asset preparation | On Daytona, upload the harness login, the Pi extension, and the in-sandbox tool shim | lines 667–703 |
| Remote mount wiring | On Daytona, mount the durable cwd, the per-harness session dirs, and the agent dir over the sandbox filesystem | lines 705–802 |
| Session init and continuity/hydration | Build the session-init payload, hydrate the prior native session id from durable storage, resume-by-id or cold-create the session | lines 804–848, 901–1014 |
| Model + capability resolution | Probe what the harness supports, assert required capabilities, resolve the requested model | lines 850–879, 1016–1036 |
| Listener attachment | Attach the session-lifetime event and permission listeners that demux into the active turn | lines 1038–1049 |

On top of those eight jobs, the function defines its own **teardown closure** (a *closure* is a
function defined inside another function that can still read the outer function's variables;
`environment.destroy`, lines 288–366) and five more closures for local remount-after-disconnect
recovery (lines 368–563), all before the real work even begins in the `try` block at line 565.

Three things make this shape costly, and they are worth spelling out because they are the whole
argument for the roadmap:

1. **Every new feature threads through this one function.** PR #5383 is the illustration: adding
   client tools on Daytona meant editing the asset-preparation job (line 691, upload the full
   tool set) and reading through the mount, session, and MCP jobs to be sure the change was safe.
   The blast radius of any change is the whole function.

2. **Testing means mocking the world.** Because the eight jobs share one function and one mutable
   object, there is no way to test "does sandbox acquisition fall back to a fresh create when
   reconnect throws?" without standing up fake versions of the mount signer, the provider, the
   workspace preparer, the capability probe, the persist driver, and the session. The dependency
   injection is already there (the `deps` argument lets a test pass fakes), but the unit under
   test is the entire acquire, not the one job.

3. **Two people cannot work in it at once.** A change to continuity and a change to mount wiring
   both edit the same 1,100-line function, so they collide. This is not hypothetical; the HANDOFF
   already records cross-lane friction on exactly these files.

This is the file behind your inline comment on line 712 (`environment.ts:712`, the
`if (environment.mountCreds && plan.isDaytona)` branch). Your words were: "is `pi` and `is` at
each place not very good programming... the problem is there is some sort of port adapter for
dealing with the logic, where you don't have to have if-then-else for the harness type within the
code. It does not look like the environment TS should know about that. Responsibility is all over
the place." That is exactly the diagnosis. `acquireEnvironment` asks `plan.isDaytona` or
`plan.isPi` at roughly a dozen sites, and each branch is a place where provider-specific or
harness-specific knowledge has leaked into a function that should be provider-agnostic and
harness-agnostic.

### 2.2 `environment-setup.ts` (388 lines): the setup prefix, decent but doing planning inline

Phase 1 carved `prepareEnvironmentSetup` out of the middle of `acquireEnvironment`. It runs
first and builds the immutable run plan, the daemon environment, the fail-closed flags, and the
empty accumulator object that `acquireEnvironment` then fills. This extraction was the right
move and the file is more coherent than `environment.ts`. But it still bundles several planning
concerns inline: mount signing, model-config planning for OpenAI-compatible custom providers
(lines 228–247), local Pi asset preparation with its fail-closed flags (lines 250–275), and the
construction of a 40-field accumulator object (lines 313–355). It returns a 30-key bundle by
`destructuring` (unpacking many named fields from one object in a single statement), which
`acquireEnvironment` immediately unpacks. That wide bundle is a symptom, not a bug: it is the
seam where one giant function was cut in half, and the two halves still share a lot of state.

### 2.3 `run-turn.ts` (640 lines): long but genuinely one job

`runTurn` runs a single turn against an already-acquired environment. It is long, but unlike
`acquireEnvironment` it does one thing, and its length comes from the real complexity of a turn:
run-limit deadlines, the pause controller, permission responders, the client-tool relay, the
prompt-versus-pause race, usage resolution, and continuity bookkeeping. It reads far better than
`environment.ts`. It is not a priority target. The one note is that the resume path (lines
439–483) and the fresh-prompt path share a lot of structure and could tighten later, but this is
polish, not debt.

### 2.4 `mcp.ts` (363 lines): clear, with two deliberate layers

`buildSessionMcpServers` builds the list of MCP servers (MCP is the protocol a harness uses to
discover and call tools) for a session. Its header is explicit that it holds two independent
layers that must never be merged: the internal Agenta tool channel and the user's own declared
servers. The file is well-documented and the `isDaytona` branching (lines 330–343) is the same
provider leak as `environment.ts`, but confined and explained. It is a natural client of the
future tool-delivery port (see §3.4), not a standalone cleanup target.

### 2.5 `client-tools.ts` (307 lines): the closed set you flagged as "weird"

This file holds the shared client-tool seam and, at lines 37–64, the `ClientToolPauseDisposition`
type: a closed set of three named string values, `"pi-native"`, `"cold-acknowledge"`, and
`"warm-hold"`. (A closed set like this is a *discriminated union*: a value that must be exactly
one of a fixed list of strings, and the compiler forces code to handle every case.) Your inline
comment on line 44 was: "quite weird structuring... it feels very weird to have pi-native,
cold-acknowledge, and warm-hold at this position. The right way is to have maybe a port and then
a pi version and a version for others. I wouldn't call this good code."

The critique is fair, and it is the same critique as the `environment.ts:712` one, wearing
different clothes. `ClientToolPauseDisposition` is a small, honest stand-in for a proper seam. It
puts the three pause behaviors in one exhaustive switch so that adding a fourth forces a
compile-time decision rather than silently falling through. That is a real improvement over an
`!isPi` test scattered across call sites, and it is why the closed set was introduced. But it is
still a per-harness behavior modelled as a string enum that the run plan carries, rather than a
method on a harness object that owns its own pause behavior. The clean shape you named, a port
with a Pi implementation and a non-Pi implementation, is precisely the harness and tool-delivery
ports of Phase 3. The closed set is the bridge to it, not the destination.

### 2.6 `run-plan.ts` (544 lines): a long builder, mostly fine

`buildRunPlan` turns a request into an immutable plan and holds all the fail-closed admission
gates (network policy, filesystem policy, code tools, remote tool delivery, reserved MCP names).
It is long because it is the one admission point and every gate has a named error message. It
reads well. Its `RunPlan` interface (lines 94–166) is where new booleans accumulate, and the
Phase 3 port design explicitly names "boolean accretion on the run plan" as an anti-goal, so this
file is a client of the ports rather than a cleanup target on its own.

---

## 3. The improvement path, as ordered follow-ups

The path has four parts, ordered so each one stands alone, ships behind the unit suite plus the
agent release gate, and makes the next one cheaper. The first part is new design work and is
sketched in full below; the rest consolidate designs that already exist.

### 3.1 First: split `acquireEnvironment` into named pipeline stages (NEW design)

This is the single highest-value change for the file you flagged, and it is a prerequisite for
the ports. The idea is to stop expressing acquire as one 1,100-line function with a shared
mutable object, and express it instead as a short, linear pipeline of named stages, each with an
explicit input and an explicit output.

**The state object that flows through.** Define one context object, `AcquireContext`, that holds
the immutable inputs (`request`, `deps`, `signal`, the built `plan`, the daemon `env`, the
`logger`, the timing logger) and the accumulator `environment` that stages fill in. Today that
accumulator already exists as `SessionEnvironment`; the change is that each stage receives the
context and returns a typed, named slice of new state, rather than every step reaching into one
blob and mutating whatever it likes. A stage's signature becomes the honest declaration of what
it reads and what it produces.

**The stage list.** Reading top to bottom, `acquireEnvironment` becomes this pipeline. Each stage
is one function in its own small file (or grouped, see §3.2):

| Stage | Input it reads | Output it produces |
|---|---|---|
| `planRun` (exists as `prepareEnvironmentSetup`) | `request`, `deps` | `plan`, daemon `env`, fail-closed flags, empty `environment` |
| `assertPreflight` | the fail-closed flags | throws, or nothing (pure, no I/O) |
| `mountLocalPreStart` | `plan`, mount creds | `environment.mountedCwd`, `environment.agentMountedPath` (local only) |
| `acquireSandbox` | `plan`, `env`, start options | `environment.sandbox`, `resumable`, in-flight registration |
| `deliverRemoteAssets` | `plan`, `sandbox` | `internalToolMcp` (Daytona only) |
| `wireRemoteMounts` | `plan`, `sandbox`, mount creds | remote cwd, session-dir, and agent mounts (Daytona only) |
| `materializeWorkspace` | `plan`, `sandbox` | `environment.workspace`, Pi session dir |
| `probeCapabilities` | `plan`, `sandbox` | `environment.capabilities` (and the required-capability assertion) |
| `buildToolDelivery` | `plan`, `capabilities`, `internalToolMcp` | session MCP servers, `environment.closeToolMcp` |
| `openSession` | session-init payload, continuity store, `persist` | `environment.session`, `sessionId`, `loadedFromContinuity` |
| `resolveModel` | `session`, wanted model | `environment.model` |
| `attachListeners` | `environment` | session-lifetime `onEvent` / `onPermissionRequest` wired |

The teardown closure and the local-remount closures move out of the function body into a small
`EnvironmentLifecycle` object built once from the context, so the pipeline driver reads as a
dozen named calls instead of thirteen hundred lines. A mid-pipeline failure still runs the one
idempotent teardown, exactly as today.

**What tests each stage gets.** This is the payoff. Each stage is an async function taking a
context and its injected dependencies, so each is unit-testable in isolation against fakes,
without booting the whole acquire:

- `acquireSandbox`: fake `startSandboxAgent` that throws on the reconnect id, assert it falls
  back to a fresh create and still registers the in-flight handle.
- `wireRemoteMounts`: fake `mountStorageRemote` that reports the store unreachable, assert the
  stage skips the mount and leaves the cwd unmounted rather than proceeding as if mounted.
- `openSession`: fake `persist` and `sandbox`, assert resume-by-id is attempted when an eligible
  prior session id exists and that a `resumeSession` throw degrades to a cold `createSession`.
- `assertPreflight`: pure, assert each fail-closed flag throws its named message.
- `deliverRemoteAssets`: fake sandbox, assert the tool shim is uploaded only for the non-Pi
  Daytona tools case and that a failed upload throws (fail loud).

None of these tests exists today, because the unit under test is the whole function.

**Size and behavior guarantee.** This is an internal refactor of one file. It changes no wire
behavior and is fully gated by the existing unit suite plus new per-stage tests. It is
independent of the folder grouping and the ports, and it makes both of them dramatically easier,
because a stage is a far smaller thing to move into a subfolder or to hand a capability object
than a region of a mega-function.

### 3.2 Second: Phase 2 folder grouping (existing design, one fix first)

The HANDOFF already specifies grouping `sandbox_agent/` into `environment/`, `session/`, `turn/`,
and `tools-delivery/` subfolders, each with an `index.ts` public entry, leaving the facade,
`engine.ts`, `runtime-contracts.ts`, `runtime-policy.ts`, and `errors.ts` at the top level.

**The one blocker to clear first is `daemon.ts`.** It computes its own package root by walking
three directory levels up from its own file location
(`dirname(dirname(dirname(fileURLToPath(import.meta.url))))`). That root drives where the harness
binaries are found at runtime. Moving `daemon.ts` one level deeper silently changes the root and
breaks binary resolution in a real deployment, and the unit tests do not exercise real binary
resolution, so typecheck and the unit suite would both stay green while the runtime broke. The
fix is to make root resolution location-independent (resolve from a known anchor rather than a
fixed number of `dirname` hops) so the file can move safely. The two cross-lane blockers the
HANDOFF also named (pi-openai and the run-plan fight) have since landed, so grouping is otherwise
unblocked.

Grouping is behavior-safe and test-gated for every file except `daemon.ts`. Done after the
pipeline split, the subfolders fall out naturally: the stage files from §3.1 are the contents of
`environment/`.

### 3.3 Third: Phase 3 provider port (existing design)

`port-design.md` §Seam 1 specifies turning the scattered "is it Daytona / is it remote" checks
into one provider object per backend that *declares* its traits (is-remote, working-directory-
is-a-FUSE-mount, can-enforce-network-policy, can-inject-runtime-credentials, who-installs-the-
harness-binary, and its keepalive style), read at one dispatch point. After the pipeline split,
each provider check is confined to a single stage (`mountLocalPreStart`, `acquireSandbox`,
`wireRemoteMounts`, `deliverRemoteAssets`), so replacing each `plan.isDaytona` branch with a
capability read is a local, testable change inside one stage rather than surgery across a
mega-function. This is the first half of the answer to your `environment.ts:712` comment.

### 3.4 Fourth: Phase 3 harness and tool-delivery ports (existing design)

`port-design.md` §Seam 2 and §Seam 3 specify the harness port (credentials as env keys plus an
optional credential file, asset preparation dispatched in one place keyed on harness id, model
naming and capability flags per harness) and the tool-delivery port (the three delivery
mechanisms behind one interface, with the deliverability policy computed in one place). These two
ports are the second half of the answer to `environment.ts:712` and the whole answer to your
`client-tools.ts:44` comment. The `ClientToolPauseDisposition` closed set becomes a method on the
harness (or tool-delivery) object: Pi's implementation parks natively, the non-Pi shim
implementation writes the acknowledging answer. Adding a harness becomes "write one object,"
which is the shape you asked for.

### 3.5 In parallel: the round-2 shell fixes (existing assessment)

The round-2 assessment (`desloppify-round2-assessment.md`) already inventories the rest of the
runner, outside this engine. Those fixes are independent of the engine work and can run in
parallel: split the 1,601-line `tracing/otel.ts` into five files, extract the keepalive dispatch
out of `server.ts`, split `tools/relay.ts`, introduce a shared logger and error-formatter, hoist
the duplicated MCP request/response envelope into one file shared by both transports, unify the
tool-kind dispatch between `tools/relay.ts` and `tools/dispatch.ts`, and move `session-pool.ts`
and `session-identity.ts` into `src/sessions/`. One of these, the shared MCP envelope, connects
to the tool-delivery port in §3.4 and is best sequenced just before it.

### 3.6 Where your PR #5383 comments land in this sequence

You left seven comments on the PR. Their dispositions:

- **`environment.ts:712`** (the `is pi` / `is daytona` branching, "responsibility all over the
  place"): the central structural critique. Addressed by §3.1 (the pipeline split isolates each
  branch into one stage) and completed by §3.3 and §3.4 (the ports replace each branch with a
  capability or harness-method read). Not a quick fix; it is the spine of this roadmap.
- **`client-tools.ts:44`** ("quite weird structuring... the right way is a port and a pi version
  and an others version"): the same critique for the pause behavior. Addressed by §3.4 (the
  disposition becomes a harness/tool-delivery port method). Structural debt, not a quick fix.
- **`PLAN.md:343`** ("I think we need to solve this today, no?" on identical-argument
  correlation): a feature-QA decision, not roadmap debt. It was resolved by your own plan-review
  decision #2: the live QA must force two client tools with identical name and arguments in one
  turn, and only if a widget attaches to the wrong call do we add the shim's request id as a
  correlating hint, as its own small commit. Actionable inside the feature PR's QA, already
  decided; carry the "add the id hint only if the forced test fails" as a small optional
  follow-up.
- **`PLAN.md:353`** ("I thought we really paused... why can't we use the native pausing behavior,
  I'm lost"): a design question about why Claude on Daytona gets a written "waiting" answer
  instead of a true pause. The answer is that Claude's in-sandbox shim blocks on a relay answer
  file and has no native cross-file-relay pause channel the way Pi's extension does, so the
  acknowledging answer is the only way to end its blocking call cleanly today. The true native
  hold is the reserved `"warm-hold"` disposition, tracked as #5384, and it becomes clean to build
  once the tool-delivery port exists (§3.4). Lands there.
- **`relay.ts:574`, `client-tools.ts:55`, `dispatch.ts:54`** (the three "Reviewer note ..."
  comments): these are your own author annotations explaining and defending the design (the
  pause/teardown ordering, why the closed set lives at the client-tool boundary, why the
  dispatch throw is defensive). No action; they document intent and should stay as comments.

Two non-blocking CodeRabbit comments on the PR are unrelated to engine quality: a `revertable`
to `revertible` spelling fix in `PLAN.md`, and a suggestion to wrap relay unit-test bodies in
`try/finally` so a failed assertion still stops the relay. Both are small and belong to the
feature PR, not this roadmap.

---

## 4. TODO table

Each row is a standalone follow-up. Size is in agent-hours: S is under a day, M is one to two
days, L is more than two days. Every row is verified by the existing unit suite plus, for the
runtime-touching ones, the agent release gate.

| # | Follow-up | Scope | Size | Prerequisite | Design doc |
|---|---|---|---|---|---|
| 1 | Pipeline-stage split of `acquireEnvironment` | Refactor `environment.ts` into ~12 named stages + one `AcquireContext`; lifecycle/remount closures become a small object; add per-stage unit tests | M | none (Phase 1 shipped) | this doc §3.1 (new) |
| 2a | `daemon.ts` package-root fix | Make binary-root resolution location-independent so the file can move | S | none | HANDOFF finding 1 |
| 2b | Phase 2 folder grouping | Group `sandbox_agent/` into `environment/` `session/` `turn/` `tools-delivery/`, each with `index.ts`; re-point imports | M | 1, 2a | HANDOFF §Phase 2 |
| 3 | Phase 3 provider port | One provider object per backend declaring its traits; single dispatch; each `isDaytona` branch becomes a capability read | L | 1, 2b | port-design §Seam 1; answers `environment.ts:712` |
| 4 | Phase 3 harness port | Harness object owning credentials, asset prep, model naming; removes `isPi` branching | L | 3 | port-design §Seam 2; answers `environment.ts:712`, `client-tools.ts:44` |
| 5 | Phase 3 tool-delivery port | Three delivery mechanisms behind one interface; deliverability policy in one place; `ClientToolPauseDisposition` becomes a harness/port method | M–L | 4 | port-design §Seam 3; answers `client-tools.ts:44` |
| 6 | Warm-hold client-tool disposition (#5384) | Build the reserved native-hold pause for the non-Pi shim | M | 5 | answers `PLAN.md:353` |
| 7 | Identical-args correlation hint (optional) | Add the shim's request id as a correlating hint, only if the forced live QA shows a widget on the wrong call | S | feature QA | plan-review decision #2; answers `PLAN.md:343` |
| S1 | Split `tracing/otel.ts` into 5 files | `trace-export`, `span-attributes`, `pi-extension-tracer`, `sandbox-agent-tracer`, `startup-banner` | L | none (parallel) | round-2 §telemetry |
| S2 | Extract keepalive dispatch from `server.ts` | Move the ~600-line keepalive subsystem to `sessions/keepalive-dispatch.ts`; re-point its tests | M | none (parallel) | round-2 §server |
| S3 | Split `tools/relay.ts` | Pull filesystem adapters into `relay-host.ts`, move redaction helpers out, type `sandbox: any` | M | none (parallel) | round-2 §relay |
| S4 | Shared logger + `errText` helper | One `makeLogger(prefix)` and one `errText(err)`; delete 8 hand-rolled loggers and ~29 repeated error slices | S | none (parallel) | round-2 §duplication 1 |
| S5 | Hoist the MCP envelope | One shared request/response envelope builder for both tool-call transports | S–M | none; sequence before 5 | round-2 §duplication 2 |
| S6 | Unify tool-kind dispatch | Collapse the drifted copy in `tools/relay.ts` into `tools/dispatch.ts` | S | S5 | round-2 §duplication 3; note `dispatch.ts:54` |
| S7 | Move session pool + identity to `src/sessions/` | Relocate `session-pool.ts` and `session-identity.ts` out of the engine folder | M | 2b | round-2 §sessions-in-two-places |
| S8 | Small nits + 2 test gaps | Typed-config reads, dead `void err`, `tools/tool-mcp-http.ts` and `sessions/auth.ts` tests | S | none | round-2 §layer-one list 8–9 |

Rows 1 through 6 are the engine-quality spine and run in order. Rows S1 through S8 are the
surrounding-runner shell fixes from the round-2 assessment and run in parallel, except S5 is best
sequenced just before row 5 (the shared envelope feeds the tool-delivery port) and S7 follows the
grouping in row 2b.
