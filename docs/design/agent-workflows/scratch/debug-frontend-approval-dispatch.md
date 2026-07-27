# Frontend approvals: why card 2 never dispatches, and why reload seals it

Scope: the two frontend defects left after the approvals fix train (PR #5382), plus the
cosmetic "bash failed" badge on the UNKNOWN sentinel. All three were reproduced live on the
EE dev stack (port 8280, agent "Local Claude", Pi harness, anthropic/claude-sonnet-4-5) in
session `00bae2fb-1467-4a70-ac87-1ed17dcf898b` on 2026-07-19, and cross-checked against the
records of the earlier QA session `2bdd4407-725e-4078-9f87-9534d8f66008`. The runner side is
healthy: card 1's approval dispatched a resume, the approved call parked with the
`APPROVED_EXECUTION_RESULT_UNKNOWN` sentinel, and card 2 re-parked cleanly. Both defects are
purely frontend.

## Shared background: what the resume turn does to the message parts

Two facts about AI SDK v6 (`ai@6.0.0-beta.150`) drive both defects' shape.

First, when an auto-resume request streams in and the last message is an assistant message,
the SDK does not create a new assistant message. `createStreamingUIMessageState`
(`web/oss/node_modules/ai/dist/index.mjs`, the `process-ui-message-stream` module) reuses the
last assistant message as the streaming target, so the resume turn's chunks are applied onto
the SAME message that turn 1 built.

Second, within that continued message, tool chunks route by `toolCallId` to the EXISTING part
(`updateToolPart` and `getToolInvocation` find the part by id and mutate it in place), while a
`start-step` chunk always pushes a fresh `{type: "step-start"}` part at the TAIL of the parts
array. So after a resume, a re-parked gate's tool part keeps its turn-1 position in the array,
and the resume turn's `step-start` marker sits behind it.

Observed live, the single assistant message after turn 2 parked (before any card-2 click):

```
idx 0  step-start                       (turn 1)
idx 1  reasoning (done)
idx 2  text (done)
idx 3  tool-bash  toolu_01GDA7...YXN94P  output-error
       errorText = "APPROVED_EXECUTION_RESULT_UNKNOWN: ..."
       approval  = {id: 95f9e06a-..., approved: true}     <- card 1, answered
idx 4  tool-bash  toolu_01HLa8...g1WnPq  approval-requested
       approval  = {id: a66702e5-...}                     <- card 2, re-parked
idx 5  step-start                       (turn 2)
```

The re-parked gate (idx 4) sits BEFORE the turn-2 `step-start` (idx 5), even though its
approval request arrived AFTER it on the wire. Part position no longer reflects event order.

## Defect A: approving card 2 never sends the resume

### The broken chain

1. The user clicks Approve on card 2. The ApprovalDock resolves the card by `approval.id`
   (`web/oss/src/components/AgentChatSlice/components/ApprovalDock.tsx:41` collects
   `approvalId` from the part) and calls `handleApprovalResponse`
   (`web/oss/src/components/AgentChatSlice/AgentConversation.tsx:1037-1043`), which records
   the live marker `{kind: "approval", id: "a66702e5-..."}` and calls
   `addToolApprovalResponse`.
2. `addToolApprovalResponse` (ai/dist/index.mjs, `AbstractChat`) flips the matching part in
   the LAST message to `approval-responded` and immediately evaluates
   `sendAutomaticallyWhen`. Observed post-click state: idx 4 became
   `state: "approval-responded", approval: {id: a66702e5-..., approved: true}`. So the SDK
   DOES re-evaluate the predicate, and the marker id DOES match the part. Neither of those
   hypotheses is the failure.
3. The predicate `agentShouldResumeAfterApproval`
   (`web/packages/agenta-playground/src/state/execution/agentApprovalResume.ts`) finds the
   answered part: `lastResolvedIdx = 4` via the marker-identity match at lines 156-160.
4. It then runs the ALREADY RESUMED guard at lines 181-184:

   ```ts
   const resumedAlready = parts
       .slice(lastResolvedIdx + 1)
       .some((part) => part.type === "step-start")
   if (resumedAlready) return false
   ```

   `parts.slice(5)` contains the turn-2 `step-start` at idx 5, so `resumedAlready` is true
   and the predicate returns false. This exact evaluation was replayed in the page against
   the live React state: `{lastResolvedIdx: 4, stepStartAfter: [{j: 5}], resumedAlready: true}`.
5. No dispatch happens. Ground truth from the network log: the last
   `POST /services/agent/v0/invoke` (reqid 2707, 21:13:55Z) is card 1's resume (its SSE
   stream re-emits `tool-approval-request {approvalId: a66702e5, toolCallId: toolu_01HLa8...}`
   and card 1's UNKNOWN `tool-output-error`). After the card-2 click there was no `/invoke`
   and no interactions POST for over 60 seconds, only the routine streams/spans/mounts
   polling. The same end state (answered card 2 plus trailing `step-start`, no resume) was
   also found preserved in the QA session `2bdd4407` still mounted in the browser.

### Why the guard exists and why it is wrong here

The guard was added (commit `74ed562d`) to stop the post-resolve loop: on a COLD replay the
runner re-issues the approved tool under a fresh id, the old `approval-responded` part lingers
forever in the same message, and a position-based "did a new step start after it" check was
used to stop the predicate from re-sending after every completion. That check assumes part
position reflects event order. On a continued (warm-resumed) message it does not: the
re-parked gate is the SAME part object, updated in place at its turn-1 index, so every gate
answered after the first resume is always "behind" a `step-start` and can never dispatch.
Card 1 worked only because at click time the sole `step-start` was at idx 0, before it.

### Minimal fix

In `agentApprovalResume.ts`, do not apply the `resumedAlready` guard on the live-marker path.
When `liveInteraction` is set, the caller has just answered exactly that approval id in this
mount, and the wiring already resets the marker to null the moment a dispatch fires
(`AgentConversation.tsx:595`), after which the predicate returns false at line 153. So a
marker-matched `approval-responded` part can dispatch at most once per click and cannot loop.
Concretely: compute and honor `resumedAlready` only in the marker-less branch (the queue and
orphan checks at lines 162-172), for example by wrapping lines 178-184 in
`if (!liveInteraction) { ... }`. The marker-less path keeps the guard, which is the path the
original loop bug lived on.

## Defect B: after reload, card 2 rehydrates dead

### The record sequence that triggers it

The durable record log for the repro session (verified via
`POST /api/sessions/records/query` for `2bdd4407-...`, identical shape in `00bae2fb-...`)
holds, in replay order:

Turn 1 (`a36c716a`):

1. user `message`
2. `thought`, `message`
3. `tool_call` c1 = `toolu_01SMvgymTgKSZkk8bYj6ncLb` (README)
4. `tool_call` c2 = `toolu_01Rp1ChwCeMBasRKbqbSMYvG` (NOTES)
5. `interaction_request` i1 = `1833336f-...` for c1 (only gate 1 is requested; Pi asks serially)
6. `tool_result` c2, `output = "DEFERRED_NOT_EXECUTED: paused for another approval; ..."`,
   `isError: true` (the park terminalizes the sibling)
7. `done`

Turn 2 (`066a69bf`, the resume):

8. user `message` (the re-sent prompt is persisted again)
9. `interaction_response` i1 `approved: true` for c1
10. `interaction_request` i2 = `6302bd4c-...` for c2 (the re-park)
11. `tool_result` c1, `output = "APPROVED_EXECUTION_RESULT_UNKNOWN: ..."`, `isError: true`
12. `done`

(The turn-2 re-emit of c1's `tool_call` upserts the turn-1 row in place, so it does not appear
twice.)

### The branch that seals the card

Hydration is `transcriptToMessages`
(`web/oss/src/components/AgentChatSlice/assets/transcriptToMessages.ts`), which keys tool
parts by `toolCallId` in a transcript-wide index. Walking the sequence:

- Row 4 creates c2's part in `input-available` and indexes it.
- Row 6 hits the `tool_result` case (lines 136-148): `isError` is true, so the part becomes
  `state: "output-error"` with the DEFERRED sentinel as `errorText`. The tool_result case
  does not distinguish runner bookkeeping sentinels from real execution results.
- Row 10, the turn-2 `interaction_request` for c2, hits the guard at lines 177-181:

  ```ts
  // Only park if still unsettled — a later `tool_result` overwrites this.
  if (part.state === "input-available") {
      part.state = "approval-requested"
      part.approval = {id: str(payload.id)}
  }
  ```

  c2's part is `output-error`, not `input-available`, so the re-park is silently dropped.
  The part keeps turn 1's DEFERRED result and never gets an `approval` field.

Observed hydrated state after reloading the wedged session with its local cache cleared:

```
message 2 (assistant):
  i 2  tool-Bash  ...YXN94P  output-error  errorText "APPROVED_EXECUTION_RESULT_UNKNOWN: ..."
       approval {id: 95f9e06a-..., approved: true}
  i 3  tool-bash  ...g1WnPq  output-error  errorText "DEFERRED_NOT_EXECUTED: paused for ..."
       (no approval field)
message 3 (user): the duplicated turn-2 user row
```

No part is `approval-requested`, so `getPendingApprovals` (ApprovalDock.tsx:32-44) returns
nothing, no dock renders, and the turn is permanently uncompletable. The same dead state also
overwrites a healthy local cache: the SWR revalidate-on-open effect
(`AgentConversation.tsx:961-982`) adopts the server transcript whenever it has MORE messages,
and the duplicated turn-2 user row guarantees it does.

### Minimal fix

In the `interaction_request` case, let a later approval request supersede a SENTINEL-ONLY
result while never downgrading a real one. Recognize sentinels by their prefixes
(`DEFERRED_NOT_EXECUTED` and `APPROVED_EXECUTION_RESULT_UNKNOWN`, defined in
`services/runner/src/tracing/otel.ts:64-69`; the web app cannot import the runner package, so
mirror them as exported constants next to the transcript adapter and reuse them in
ToolActivity, which today hard-codes the deferred prefix at line 38). Concretely, replace the
`input-available` check with: park when `part.state === "input-available"`, or when
`part.state === "output-error"` and `part.errorText` starts with a sentinel prefix; when
re-parking, clear `errorText`/`output` and set `state: "approval-requested"` plus the new
`approval.id`. A real result (`output-available`, `output-denied`, or a non-sentinel
`output-error`) still wins over a stale request row, which preserves the existing "a later
tool_result overwrites this" invariant for genuinely executed calls.

## The badge: UNKNOWN sentinel renders as a red "failed"

`ToolActivity.tsx` special-cases only the deferred sentinel: `isDeferredError` (lines 38-40)
matches the `DEFERRED_NOT_EXECUTED:` prefix and renders the neutral clock icon and "waiting
on another approval". The UNKNOWN sentinel takes the generic `output-error` path instead:

- `rowSummary` line 88 returns "failed",
- `StatusIcon` lines 102-105 renders the red `Warning` icon,
- the mid-text gets `type="danger"` at line 210,
- and the collapsed group summary counts it in `failed` (lines 351-361), producing the red
  "Bash failed" / "N failed" badge.

Fix in one move: add an `isUnknownResultError` (prefix `APPROVED_EXECUTION_RESULT_UNKNOWN`)
beside `isDeferredError`, treat it as neutral in all four spots (icon, row summary such as
"approved, result unknown", non-danger text, excluded from the `failed` count), sourcing both
prefixes from the shared constants introduced by the defect-B fix. This applies to live
rendering too, not just replay: the live turn-2 stream delivers the same sentinel as a
`tool-output-error` chunk, and the transcript showed the red "bash failed" before any reload.

## Repro notes

- Prompt: "Append the line \"hello from QA\" to agent-files/README.md and to
  agent-files/NOTES.md, as two separate Bash commands issued in parallel in the same turn."
- Card 1 approval id `95f9e06a-...` dispatched `/invoke` (reqid 2707); card 2 approval id
  `a66702e5-...` clicked afterward produced zero network traffic beyond polling.
- The predicate evaluation was replayed in-page against the live `useChat` state via the
  React fiber (component `AgentConversation`, the messages hook), matching the code path
  exactly: marker found the part (`lastResolvedIdx = 4`), `step-start` at index 5 vetoed it.
- Defect B was forced deterministically by deleting the session's entry from the
  `agenta:agent-chat:messages` localStorage map and reloading, which routes hydration through
  `loadSessionMessages` and `transcriptToMessages`.
