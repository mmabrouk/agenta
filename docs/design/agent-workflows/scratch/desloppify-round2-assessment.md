# Desloppify round two: what about the two layers we did not touch?

**Date:** 2026-07-18
**Author:** staff-level code assessment (read-only)
**Method:** the desloppify skill set on the local branch `chore/desloppify-code-skill`
(commit `42db761f7e`). I applied its scoring dimensions — cohesion, naming, dead code,
duplicated logic, error handling, type safety, test coverage, and cross-module
architecture — rather than inventing my own.

## Why this assessment exists

We just finished a desloppify round on one piece of code: the "sandbox agent engine"
inside the runner. That engine used to be a single 1,500-line file that did everything.
We broke it into about ten small files, each with one job, and left behind a 44-line
front-door file that only points at them.

A note on one word you will see throughout. In this codebase a *facade* is a small file
whose entire job is to re-export names from other files, so that everything outside has a
single, stable place to import from and never has to know how the work is split up
internally. `services/runner/src/engines/sandbox_agent.ts` is that front door; the real
work lives in the sibling folder `sandbox_agent/`.

Mahmoud's question is about the two layers we did **not** clean up:

- **Layer one — the rest of the TypeScript runner.** Everything under
  `services/runner/src/` that is not the sandbox-agent engine: the web server, the tool
  relay, session bookkeeping, telemetry, configuration, and the command-line entrypoint.
  This is a Node service, meaning a long-running program written in TypeScript (a typed
  dialect of JavaScript) that answers HTTP requests.
- **Layer two — the Python agent service.** The small Python program that exposes the
  `/invoke` endpoint the playground calls, and that in turn talks to the runner.

For each layer the decision Mahmoud actually needs is one of three: (a) it needs a full
desloppify round like the engine got; (b) it needs a handful of targeted fixes that I
name file by file; or (c) it is fine as it is.

---

## The short answer

| Layer | Verdict | One-line reason |
|---|---|---|
| **1. TypeScript runner** | **(b) targeted fixes — a sizeable list, but not a full round** | The code is disciplined and very well tested, but two files are genuine "does-too-many-things" modules and there are three real duplication themes. |
| **2. Python agent service** | **(c) fine as it is** | It is 462 lines of thin, well-documented wiring over the SDK, with 2,000+ lines of tests. Nothing to clean up beyond deleting a few back-compat shims later. |

The rest of this document is the evidence.

---

# Layer one: the TypeScript runner

## What this layer is and what state it is in

The runner is the program that actually drives a coding agent inside a sandbox. When a run
comes in, it acquires a sandbox, wires up the tools the agent may call, streams the agent's
output back, records telemetry, and keeps the session alive between turns. Excluding the
sandbox-agent engine we already cleaned, this layer is about 20 files and roughly 9,000
lines of TypeScript.

The first thing to say is that this is **unusually disciplined code**, and the desloppify
lens confirms it rather than the reverse. Across the whole layer there are essentially no
`TODO`/`FIXME` markers, no `@ts-ignore` (a comment that switches off type checking for the
next line), no swallowed errors hiding real failures, and no hardcoded external URLs beyond
documented in-network defaults. Error handling follows one clear house style: catch the
error, log the full detail to the server's own log, put only a short safe message on the
wire, and fail closed (deny rather than allow) when a security decision cannot be made.
Constant-time password comparison, request-body size caps, and protection against
malicious regular expressions are all present. This is not sloppy code.

The test coverage is genuinely strong. Of the roughly 20 source files, only two tiny
helpers (a 47-line ID generator and an 18-line pair of environment-variable names) have no
dedicated test, and both are exercised indirectly. There are about 75 test files totalling
nearly 30,000 lines. So any cleanup here is a **low-risk, test-backed refactor**, not a
rewrite in the dark.

So why is the verdict "targeted fixes" and not "fine as it is"? Two reasons: two files
have grown into "god-modules," and three kinds of duplication have spread across the layer.

A *god-module*, in the desloppify vocabulary, is a file that trips two independent alarms
at once: it is large **and** it does several unrelated jobs. Size alone does not count —
a long file that does one thing well is fine. The two files below trip both alarms.

## The single biggest cleanup target: the telemetry file

`services/runner/src/tracing/otel.ts` is 1,601 lines and is the largest file in the entire
runner. Its job is telemetry: turning an agent's event stream into a tree of timed spans
and shipping them to Agenta's observability backend. (OpenTelemetry, abbreviated OTel, is
the industry-standard library for this; OTLP is its wire format.)

This is a textbook god-module. Inside that one file live at least five genuinely
independent jobs:

1. The shared export machinery: the registries, the exporter cache, the batch processor,
   and the flush logic.
2. The attribute mapping: pure functions that translate a chat message into span
   attributes.
3. One complete tracer for the in-process Pi harness.
4. A second complete tracer, about 500 lines on its own, driven by the sandbox agent's
   events.
5. **Startup-banner text stripping** — three functions that strip the harness's startup
   banner out of its text output. This has nothing whatsoever to do with telemetry, and
   it already has its own separate test file. It is simply sitting in the wrong file.

The natural split is five files: `trace-export.ts`, `span-attributes.ts`,
`pi-extension-tracer.ts`, `sandbox-agent-tracer.ts`, and `startup-banner.ts`. Every seam is
clean and the existing tests already cover each piece, so this is mechanical work.

Two smaller notes on the same file. It uses TypeScript's `any` type — the escape hatch that
turns off all type checking for a value — in 30 places, because the incoming harness events
are loosely typed. Most of that could be recovered with a few small event interfaces. And
there is one dead field, `RunConfig.baggage`, that is written in one place and never read
anywhere; it should be removed or wired up.

**Verdict for this file specifically: it warrants the same treatment the engine got.** It
is the one file in the runner that, on its own, would benefit from a focused desloppify pass.

## The second god-module: the web server

`services/runner/src/server.ts` is 1,271 lines. Its header comment says it is the HTTP
transport — the code that accepts requests and returns responses. But it is actually two
things bolted together. About 600 of its lines are the *keepalive dispatch engine*: the
subsystem that keeps a session's sandbox warm between turns, parks a run that is waiting for
an approval, and resumes it later. The strongest evidence that this is a separate subsystem
is that two of the test files already import it and test it on its own, as if it were a
standalone module — the tests treat it as separate even though the code does not.

The fix is one clean extraction: move the keepalive subsystem into its own file (for
example `sessions/keepalive-dispatch.ts`) and re-point those tests at it. That leaves
`server.ts` as roughly 400 lines of honest HTTP plumbing, which is what its own comment
claims it is. There is also one 440-line function inside it whose two branches — resuming an
idle session and resuming after an approval — are near-duplicates that a shared helper would
collapse.

## The one god-module in the tool subsystem

The `tools/` folder (the tool relay and MCP bridge — the machinery that lets the agent
inside the sandbox call tools out in the world) is, on the whole, well factored. It has real
shared cores — a wire-protocol file, a schema-accessor file, a callback transport — and the
file headers narrate the history of every extraction. The family is healthy.

The exception is `tools/relay.ts` at 748 lines, which carries four unrelated jobs: the
filesystem adapters that read and write relay files, the tool-execution dispatch, the poll
loop itself, and a set of argument-redaction helpers. Splitting the filesystem adapters into
`relay-host.ts` and moving the redaction helpers out leaves `relay.ts` as just the loop.

## Three duplication themes that run across the whole layer

Beyond the individual files, three kinds of copy-paste have spread across the runner. None
is dangerous today; all will multiply as the runner grows.

1. **Every file writes its own logger.** Eight files in this layer each define their own
   little function that writes a prefixed line to standard error, and 29 separate places
   repeat the exact idiom `err instanceof Error ? err.message : err` to extract an error
   message. There is no shared logging or error-formatting module. One `makeLogger(prefix)`
   helper and one `errText(err)` helper would delete dozens of copies.

2. **The tool-call protocol handler is written twice.** The runner speaks MCP (the protocol
   a harness uses to discover and call tools) over two transports: an HTTP server for the
   local Claude harness, and a standard-input/output shim uploaded into the sandbox. The two
   files build the identical request-and-response envelopes; they differ only at the single
   step where one runs the tool in-process and the other writes a relay file. The stdio file
   even says in its header "Mirrors the HTTP one." The envelope building is safe to share.

3. **The tool-kind dispatch is written twice.** `tools/dispatch.ts` claims in its header to
   own tool dispatch "once," but `tools/relay.ts` re-implements the same branch and has
   already drifted — the relay copy handles a direct-call case that the shared copy does not.
   These should be unified so the direct-call branch is not stranded.

## One structural oddity worth naming: sessions live in two places

Session logic is split across two directories. `src/sessions/` holds the durable
persistence, the liveness heartbeat, and the interaction lifecycle. But the session *pool*,
the session *identity*, and session *continuity* live inside
`src/engines/sandbox_agent/` — physically inside the engine folder we just cleaned. The web
server reaches past the engine's front door and imports the pool and identity files
directly.

This is worth understanding precisely, because it is the answer to the "does anything reach
around the facade" question. The engine's front door (`sandbox_agent.ts`) deliberately
re-exports only the run-driving surface. The session pool and identity files are not part of
that surface, so the server importing them directly is not a violation of the facade — it is
a sign that **those files are filed in the wrong folder**. They are runner-wide session
infrastructure that happens to sit under the engine directory. Moving `session-pool.ts` and
`session-identity.ts` up into `src/sessions/` would put all session code in one place and
remove the reach-around entirely.

## Can this layer absorb the growth that is coming?

The runner is about to grow more sandbox backends (E2B, Docker) and more harnesses (Codex,
OpenCode), and may run several instances in parallel. The desloppify lens asks whether the
current shape helps or fights that.

**On new sandbox backends, the structure is genuinely good.** There is a clean interface,
`RelayHost`, and adding a backend means implementing it and registering a factory. The two
existing implementations prove the pattern. The one friction point is that the sandbox
handle is typed as `any`, so the compiler will not tell a Docker or E2B author which methods
they must provide. Typing that contract now would make the third backend cheap.

**On new harnesses, it mostly absorbs, but one seam fights.** Harness delivery already forks
cleanly between the two harness classes. The friction is duplication theme two above: every
new tool-call *transport* copies the protocol envelope a third time, and each copy is a place
where a subtle bug could regress independently. Fixing that duplication before the next
harness lands is the highest-leverage move for growth.

## Layer-one verdict, stated plainly

**(b) A sizeable but bounded set of targeted fixes — not a full desloppify round.** The
engine needed a full round because it was one enormous file doing everything. This layer is
already mostly well-modularised, consistently styled, and heavily tested, so running the full
blind-review-and-triage cycle across all 20 files would find little. What it needs is a named
list:

1. Split `tracing/otel.ts` into five files. *(This one file alone deserves the engine
   treatment.)*
2. Extract the keepalive dispatch engine out of `server.ts` and re-point its tests.
3. Split the filesystem adapters and redaction helpers out of `tools/relay.ts`, and type the
   `sandbox: any` contract.
4. Introduce a shared logger and error-formatter; delete the eight hand-rolled loggers and
   29 repeated error-slice lines.
5. Hoist the MCP protocol envelope into one shared file used by both transports.
6. Unify the tool-kind dispatch between `tools/relay.ts` and `tools/dispatch.ts`.
7. Move `session-pool.ts` and `session-identity.ts` out of the engine folder into
   `src/sessions/`.
8. Small nits: `server.ts` should read the concurrency limit from the typed config instead
   of re-parsing the environment variable; remove two dead `void err;` statements; move two
   misplaced import lines to the top of their files; route the last two direct
   `process.env` reads (`permission-plan.ts`, `apiBase.ts`) through the typed config so the
   "one place reads the environment" rule holds everywhere.
9. Close two test gaps: add a dedicated test for `tools/tool-mcp-http.ts` (it is a security
   boundary — bearer auth on a loopback server — yet only covered indirectly), and a test for
   `sessions/auth.ts`'s credential refresh (the one wholly uncovered function in the layer).

### Effort and how to verify

The whole list is roughly **15 to 21 agent-hours**, and it splits naturally into
independent pull requests (one per god-module, one for the de-duplication, one for the small
nits and test gaps). Because this touches the telemetry sink that feeds observability, the
keepalive engine that is the heart of session continuity, and the tool-execution path, unit
tests alone are not quite enough at the end: **the work warrants the agent release gate**
(the wire-level end-to-end QA harness) as a final check, not just the existing unit suites.
Each individual extraction, however, is verified by the unit tests that already exist.

---

# Layer two: the Python agent service

## What this layer is and what state it is in

This is the Python program at `services/oss/src/agent/`. It exposes the `/invoke` endpoint
the playground calls, and it hands the real work to the runner. It is 462 lines across nine
files, and the striking thing is how little it does on its own:

- `app.py` (168 lines) builds one composition object and delegates to the SDK's agent
  handler. It used to re-implement the streaming, folding, and capability-gating logic; that
  now lives in the SDK, and the service just assembles the pieces and forwards the call.
- `config.py` (119 lines) reads the on-file agent template (the AGENTS.md text, the model,
  the tools) and a few environment variables.
- `schemas.py` (80 lines) holds the JSON schemas the agent advertises so the playground
  knows to render a chat box.
- `secrets.py`, `tools/gateway.py`, `tools/secrets.py`, `tools/resolver.py` are all **thin
  re-exports** of the SDK's platform package. Their entire body is "import this from the SDK
  under the old name so existing imports keep working."

In other words, almost all the real logic lives one layer down in the Python SDK
(`sdks/python`), which is a different codebase and outside the scope of this question. This
service is a composition-and-forwarding layer by design.

## Quality against the desloppify dimensions and the API conventions

Measured honestly, this layer is clean. There are no swallowed exceptions, no `type: ignore`
markers, no `TODO`/`FIXME`, and no oversized functions. Every module carries a thorough
docstring that explains not just what it does but why it is shaped that way. The tests are
generous: about 2,056 lines of tests for 462 lines of code, including a 749-line test of the
invoke handler.

I also judged it against `api/AGENTS.md`, the backend's own convention document, since the
prompt asked for that. Two points are worth stating:

- The convention says application configuration should be centralised and not read via
  scattered `os.getenv` calls. This service reads exactly three environment variables, and
  all three live in `config.py` behind named accessor functions (`runner_dir`, `runner_url`,
  `config_dir`). That is the spirit of the rule, not a violation of it — the reads are
  centralised in the config module and consumed as functions.
- The convention says a service should raise its own domain exceptions rather than leak
  lower-level ones. `select_backend` does exactly this: it raises `LocalSandboxNotAllowedError`
  when a deployment refuses the local sandbox, rather than letting a raw failure escape.

## Can this layer absorb the growth that is coming?

Yes, almost for free. More sandbox backends and more harnesses barely touch this file,
because it forwards the sandbox, harness, and tool fields to the SDK and the runner rather
than switching on them itself. Adding E2B or Docker means adding a name to the enabled-provider
set, which is configuration, not code here. This is the least-affected layer in the whole
system.

## Layer-two verdict, stated plainly

**(c) Fine as it is.** There is no cleanup this layer needs. The only forward-looking note is
that a handful of the thin re-export files (`secrets.py`, `tools/gateway.py`,
`tools/secrets.py`) exist only to keep one deprecated integration test importing under the old
names. When that test (`test_resolve_secrets_http.py`) is finally removed, those shim files can
be deleted too. That is optional tidying, roughly **one agent-hour**, verified by the existing
unit tests, and it needs no release gate.

---

# The two decisions on one page

**Layer one, the TypeScript runner: (b) targeted fixes, a sizeable list.** The code is
disciplined and well tested, so it does not need a full desloppify round — but it does need
a named set of extractions and de-duplications, led by splitting the 1,601-line telemetry
file and pulling the keepalive engine out of the web server. Roughly 15 to 21 agent-hours,
best done as several small pull requests, finished with a run of the agent release gate.

**Layer two, the Python agent service: (c) fine as it is.** It is thin, well-documented,
and heavily tested wiring over the SDK, and it absorbs the coming growth without change.
The only optional work is deleting a few back-compat shim files once a deprecated test is
removed, about one agent-hour.
