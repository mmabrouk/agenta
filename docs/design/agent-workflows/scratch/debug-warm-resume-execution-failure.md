# Root cause: warm resume never executes the approved command (2026-07-19 evening)

This report explains why, on the EE dev stack (`agenta-ee-dev-wp-b2-rendering-*`), a Pi
session with two parallel gated Bash calls approves cleanly but never executes the approved
command. The failure was live in sessions `183f6297`, `324f72f5`, and `94ed7f8a`
(19:12 to 19:27 UTC), and I reproduced it deterministically at the wire level (session
`bbc4bdf2`, 20:15 UTC). Every claim below is backed by a timestamped runner log line, a
file-and-line code citation, or a scripted reproduction run against the live stack.

## Orientation: the moving parts

- **The runner** is the Node sidecar under `services/runner/`. Per environment it spawns a
  **local sandbox daemon**: the `sandbox-agent server --host 127.0.0.1 --port <random>`
  binary from the `@sandbox-agent/cli-linux-x64` package (resolved in
  `services/runner/src/engines/sandbox_agent/daemon.ts`). That daemon in turn runs
  `pi-acp` (the bridge that speaks the Agent Client Protocol, ACP, over HTTP) and the `pi`
  harness process. "ACP write" in the logs is the runner POSTing a JSON-RPC message to that
  daemon's localhost port.
- **A gate** is a human-approval request. Our Pi extension
  (`services/runner/src/extensions/agenta.ts:224`) registers a `tool_call` hook that calls
  `ctx.ui.confirm` and waits; pi-acp surfaces the confirm as an ACP
  `session/request_permission`, which the runner shows as an approval card.
- **Park and warm resume.** An unanswered gate ends the turn and parks the live harness
  session in a keepalive pool. When the answer arrives, the runner checks the session back
  out and answers the still-pending permission request in place
  (`run-turn.ts:637`, `respondPermission`). That is a warm resume.
- **The bounded closure wait** is new since this afternoon (fix-train step 2, commit
  `4c8c809984`, authored 14:52 UTC). When a resume turn ends paused, the runner now holds
  the turn open until every approved call reports a terminal frame, bounded per call by the
  per-tool-call run limit (`run-turn.ts:675-689`, calling `waitForToolCallClosure` at
  `run-turn.ts:225-250`). The bound is `resolvedRunLimits.toolCallMs`, whose default is
  `DEFAULT_TOOL_CALL_TIMEOUT_MS = 5 * 60_000` (five minutes, `run-limits.ts:40`).

## The root cause in one paragraph

Pi's engine does not execute a parallel tool call the moment its approval resolves. In
`pi-agent-core`'s `executeToolCallsParallel` (in the installed package,
`dist/agent-loop.js` around lines 332-365), the preparation loop is sequential: it awaits
`config.beforeToolCall` for call 1 (that await is our confirm dialog, `agent-loop.js:405`),
pushes call 1's execution as a deferred thunk, then awaits `beforeToolCall` for call 2, and
only after the loop finishes does `Promise.all` run the execution thunks. So the approved
first command cannot start until the second command's confirm is also answered. The new
bounded closure wait assumes the opposite: it holds the resume turn open until the approved
call closes, which means the turn does not park, the second approval card does not reach a
settled turn the user can answer, and nothing can ever satisfy the wait. The two sides
deadlock, bounded only by the five-minute per-call limit. During those five minutes the
user sees "bash running", the Python agent service's stream watchdog eventually kills the
client stream with a generic "agent run failed", and no file is ever written.

## The verified timeline (session 183f6297, all times UTC)

1. **19:12:39** turn 1 starts cold; the environment acquires a fresh daemon on
   `127.0.0.1:41765`. Pi raises the confirm for the first Bash call (`toolu_012Z3...`,
   append to `agent-files/README.md`). At **19:12:48.562** the runner parks
   (`state=awaiting_approval ttl=300000ms`) and the turn ends normally.
2. **19:13:16.046** the user's approval arrives:
   `[keepalive] resume ... gates=1 answered=1 carried=0 approve=1`. One millisecond later
   the runner answers the parked gate (`resume answered gate reply=once`), and at
   **19:13:16.050** Pi raises the confirm for the second Bash call (`toolu_01JWz...`).
   This is Pi's preparation loop advancing: answer for call 1 received, hook for call 2 now
   awaited. Per the engine code above, call 1's execution has NOT started and will not
   start until this second confirm resolves.
3. **19:13:16 to 19:18:16** nothing happens. The turn's heartbeats report `running=true`
   every 30 seconds. The geesefs mount statistics show `0.00 write/s` for the entire
   window, which proves the append never executed (the sandbox working directory is the
   mount). No frame of any kind arrives for the approved call, so the bounded closure wait
   cannot finish early.
4. **19:18:16.051**, exactly 300.001 seconds after the resume, the wait expires. The
   runner settles the approved call with the `APPROVED_EXECUTION_RESULT_UNKNOWN` sentinel
   (`tracing/otel.ts:68`), logs `prompt stopReason=paused`, re-parks with the second gate
   pending (`(re-park) poolSize=3`, the observed pool growth), and persists
   `tool_result` and `done`.
5. **19:21:26.760** the runner process receives SIGTERM: the dev container's `tsx watch`
   restarted the server because source files changed (another agent was committing to the
   working tree; the rebuild log `[build-extension] wrote dist/extensions/agenta.js`
   follows at 19:21:32). The shutdown runs `[keepalive] destroyAll count=2`, killing the
   parked daemons. The queued ACP writes to those daemons then fail: first
   `ECONNRESET`/`other side closed` on in-flight sockets, then
   `connect ECONNREFUSED 127.0.0.1:41765` and `127.0.0.1:33601` on retry, ending in
   `ACP write error` and `[sandbox-agent] unhandledRejection`. Port 41765 is session
   183f6297's daemon and port 33601 is session 324f72f5's daemon; both had just been
   deliberately killed. After this point the parked sessions are gone, so the user's
   second approval can only take the cold path against a fresh sandbox, and the two
   commands never run at all.

Sessions `324f72f5` (resume 19:18:23, hang) and `94ed7f8a` (resume 19:22:16.035, second
gate at 19:22:16.039, re-park at 19:27:16.040, again exactly plus 300 seconds) show the
identical pattern.

### The control that proves the regression

The same box ran the same shape correctly this morning on pre-train code. In session
`db58551b` (see `debug-concurrent-approvals-db58551b.md`), the resume at 09:53:22.458
answered gate 1, Pi raised gate 2 at 09:53:22.460, and the runner logged
`prompt stopReason=paused` and re-parked at **09:53:22.461**: three milliseconds, not five
minutes, because the pre-train code parked immediately instead of waiting for the approved
call to close. Both commands then executed once the last gate was answered, which is
exactly the engine contract described above. The morning log lines also carry the old
resume format (`gates=1 approve=1`, no `answered=`/`carried=` counters), confirming the
code difference. The overnight wire-level chain runs at 00:01 and 00:06 UTC
(`debug/qa-concurrent-approvals/chain_bash.json`, `chain_pi.json`) passed on that same
pre-train code and show the approved Pi call executing only on the turn after the LAST
gate was answered, which independently confirms the engine's batching behavior.

## Reproduction (question 4)

Both runs were made against the live stack at 20:12 to 20:20 UTC with the existing
wire-level driver (`debug/qa-concurrent-approvals/multi_gate.py`, Pi cell C3, local
sandbox).

- **Single gate, approve: PASSES.** Session `65f26933`: warm resume answered the gate at
  20:13:05.494 and the prompt completed at 20:13:07.847 (`stopReason=end_turn`), with the
  command executed and its output streamed (`tool-output-available`, reply "WROTE",
  2.4 seconds). With no sibling confirm pending, Pi's preparation loop finishes and the
  batch executes immediately, so the closure wait is satisfied and never visibly delays
  anything.
- **Two parallel gated calls: FAILS exactly like the live sessions.** Session `bbc4bdf2`:
  turn 1 parked on gate A; the resume answered gate A at 20:15:02.090; Pi raised gate B at
  20:15:02.096; the turn then hung with `running=true` heartbeats; the product endpoint's
  SSE stream errored with `"agent run failed"` (the agent service watchdog, at roughly
  three minutes); the runner re-parked at **20:20:02.100**, 300.010 seconds after the
  resume; neither `/tmp/qa-a-*.txt` nor `/tmp/qa-b-*.txt` was ever created. This repro
  used the OpenAI model (`call_...` tool ids), while the live failures used Anthropic
  (`toolu_...` ids), so the failure is the two-call shape, not the model.

The single-gate pass plus the two-call fail narrows the defect to the train's park path,
exactly as the question anticipated.

## Answers to the five questions

**1. What is the process on 127.0.0.1:port, and why ECONNREFUSED?** It is the per-environment
local sandbox daemon: `sandbox-agent server --host 127.0.0.1 --port <random> --token ...`
(binary from `@sandbox-agent/cli-linux-x64`, spawned by the runner's local provider; the
process tree inside the container is sandbox-agent, then pi-acp, then pi). The resume is
NOT targeting a stale port: every acquire and resume in the logs used the daemon's current
live port. The ECONNREFUSED writes are teardown noise with two triggers, both of which
kill a daemon while an ACP cancel or notification for it is still queued: (a) keepalive
eviction on `approval-ttl-expire` (a background loop on this box parks a "gh pr list"
approval every five minutes that nobody answers, so this fires regularly), and (b) the
19:21:26 SIGTERM `destroyAll` when the dev watcher restarted the runner because the
working tree changed. The `unhandledRejection` is the unawaited retry chain surfacing.
No daemon crashed on its own and there are no orphan daemons. The engine-quality
roadmap's daemon weakness (the `daemon.ts` package-root derivation) is about binary
resolution during refactors and is not implicated.

**2. Is this a regression from the fix train?** Yes: step 2, commit `4c8c809984`
("only real executor evidence settles a paused turn's tool calls"), authored 14:52 UTC,
between the morning success and the evening failures. It added the bounded closure wait at
`run-turn.ts:675-689`. The released main's client-tools and deny-frame changes are not
implicated: the daemon-side packages (`pi-acp`, `sandbox-agent`, the Pi extension) have no
newer commits than the morning code, and the single-gate warm resume works on the current
tree. Step 4's carried-forward re-park is a bystander here: every failing resume logged
`carried=0`, and the second gate was a fresh gate, not a carried one. The pause
controller's park-mode destroy skip (`run-turn.ts:196`) behaved correctly; the sessions
stayed alive until the watcher restart killed them.

**3. Why does fusermount fail, and is geesefs implicated?** Red herring. The
`fusermount -uz` failure at mount time is the pre-clean step of `mountStorage` running
against a path that is not currently mounted; the code then confirms the path is gone
("unmounted (confirmed gone)") and mounts fresh ("verified alive"). The same benign
pattern appears on every acquire, including this morning's healthy sessions. During the
hang the geesefs mounts were alive and their statistics logs kept printing; the
`0.00 write/s` readings are evidence that the command never ran, not that the mount was
broken.

**4. Minimal reproduction.** Done, see above: single-gate warm resume executes (PASS);
the two-call shape hangs for exactly the five-minute per-call limit and executes nothing
(FAIL). Attribution therefore lands on the train's park path, not on the environment.

**5. The defective code path and the minimal fix.** The defect is the unconditional
bounded closure wait at `services/runner/src/engines/sandbox_agent/run-turn.ts:675-689`
interacting with Pi's batched parallel execution
(`pi-agent-core/dist/agent-loop.js`, `executeToolCallsParallel`: sequential
`beforeToolCall` awaits, execution only after all confirms resolve). The wait's premise,
"an approved call is executing and will close on its own", is false on Pi whenever the
same resume turn surfaced a NEW unanswered gate. The minimal correction: skip the bounded
wait and settle the approved calls with `APPROVED_EXECUTION_RESULT_UNKNOWN` immediately
when the turn is a Pi turn and a fresh parkable gate is pending
(`plan.isPi && opts.approvalParkMode && env.parkedApprovals.size > 0` at that point in
`runTurn`), so the turn parks in milliseconds and the new card reaches the user; the
approved command then executes when the final sibling gate is answered, which is Pi's
engine contract and is what the morning and overnight runs did. Keep the wait for Claude:
the overnight `chain_bash.json` turn 2 shows Claude delivering the approved call's real
result inside the resume turn, which is precisely the case the wait was built for. A
follow-up worth considering upstream is asking Pi for per-call execution as approvals
resolve, but that is an engine change, not this fix.

## Attribution

The execution failure and five-minute hang are the fix train (step 2's bounded wait), not
the released main and not the infrastructure. Two environmental effects compounded the
user experience on this box tonight but did not cause the failure: the dev watcher
restarting the runner mid-incident destroyed the parked sessions (producing the
ECONNREFUSED and unhandledRejection noise and making the second approvals land cold), and
the agent service's stream watchdog turned the silent five-minute wait into a generic
"agent run failed" on the wire. The reported "bash not handled by this client" message was
not found in the runner source, the daemon packages, or tonight's runner logs; it is
consistent with the post-restart cold path and needs a session record to pin down, but
nothing in this incident's root cause depends on it.

## Follow-ups

- Fix the wait as above and add the missing regression shape to the runner tests: two
  parallel Pi gates, approve the first, assert the resume turn parks within a bounded
  short interval and that both commands execute after the second approval.
- The five-minute silence also shows a monitoring gap: a resume turn that emits no frame
  for minutes while holding a checked-out session should log the wait it is in.
- The dev stack's watcher restart during a live incident destroyed the evidence sessions.
  When debugging parked-session behavior on this box, freeze commits to the tree or run
  the runner outside watch mode.
