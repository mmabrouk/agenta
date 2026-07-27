# Self-hosting hardening plan

## What this is

Two things landed at once and point at the same area:

1. GitHub issue [#5401](https://github.com/Agenta-AI/agenta/issues/5401): a self-hoster
   followed the quick-start guide and `docker compose up` failed because
   `ghcr.io/agenta-ai/agenta-runner:latest` does not exist on the registry.
2. Field feedback from a second self-hoster who hit a string of problems setting up the OSS
   stack with the agent runtime.

Every item below was verified against the current code on 2026-07-20, not taken on trust
from the reports. The feedback was about a week old, so several bugs were already fixed in
the meantime. Each item says whether it is still real, already fixed, or by design, with the
file and line that decides it.

The items are sorted into three groups:

- **Part 1:** problems that belong to self-hosting and that we will fix.
- **Part 2:** problems that are already fixed in code, where the only remaining issue is a
  self-hoster running an old runner image.
- **Part 3:** separate bugs that are not about self-hosting and belong to other teams.

Open decisions that need your call are collected at the end, under "Decisions for you."

---

## Part 1: Self-hosting problems we will fix

### 1. Runner image `agenta-runner:latest` is never published (issue #5401)

**Status: still present. Confirmed.**

One workflow in the private `agenta_cloud` repo, `43-release-to-ghcr.yml`, is the only thing
that builds the public OSS images and writes the `latest` tag. It has two job lists
(`build-internal` at lines 58-61 and `merge-and-mirror` at lines 162-165), and both name
only `agenta-api`, `agenta-web`, and `agenta-services`. The runner was never added, so it is
never built for the public registry and never tagged. The build script and image-naming
logic already handle the runner, and the runner's Dockerfile
(`services/runner/docker/Dockerfile.gh`) exists and is tracked, so nothing else in the
pipeline needs to change.

The mirror step already writes both the version tag and `latest` in one call, so adding the
runner publishes `agenta-runner:vX.Y.Z` (which Helm pulls) and `agenta-runner:latest` (which
Docker Compose pulls) together.

The `agenta-completion` and `agenta-chat` images that look like a missing pattern are old
leftovers from before those services merged into `agenta-services`. Nothing republishes them.
They are a red herring.

**The fix is two lines: add `agenta-runner` to both job lists.** That is the whole change.
There is no error in our CI today, because the workflow does not touch the runner at all
right now. The 404 the user saw happens later, on their own `docker compose up`, not inside
our pipeline. The image publishes on the next release, which is expected within hours. We are
not doing a one-time manual publish; waiting for the next release is fine.

(We are not adding any guard or extra safety step to the workflow. One edge case exists in
theory: if a future release branch were missing the runner's build file, the build step would
quietly pass while the publish step failed with an unclear message. That build file is
permanently tracked and always included, so the case does not arise. If we ever want a clearer
message for it, we can add one later; it is not needed now.)

**Areas:** CI only. Compose and Helm need no change; both start working the moment the image
exists.

---

### 2. Empty `AGENTA_STORE_REGION` makes agent runs hang forever

**Status: still present. Confirmed. Decided by `api/oss/src/utils/env.py:1093`.**

**Context first: what the "store" is, and why even a local setup needs a region.**
Agenta has an object store, which is an S3 bucket that holds each agent run's durable
working directory. When an agent runs, it works inside a throwaway sandbox, but it needs a
folder that survives across turns and across sandboxes. That folder lives in the store and is
mounted into the sandbox over the network. If the store cannot answer, the run cannot get its
working folder, so it hangs. That is why this bug freezes runs rather than breaking some minor
feature.

The setting is called "store region," not "S3 region," on purpose. The store is an
abstraction that runs on different S3-compatible backends: a bundled open-source server called
SeaweedFS for local self-host, and real Amazon S3 or Cloudflare R2 in cloud. All of them speak
the S3 protocol through one code path (`api/oss/src/core/store/storage.py:55-61`,
`api/oss/src/utils/env.py:1082`). This matters for the common belief that "local self-hosters
do not use S3." They do. The local backend is SeaweedFS, which is an S3 server. An S3 client
must be handed a region to sign its requests, even when the backend ignores the value. So a
local self-host needs a region too. For SeaweedFS the exact value does not matter; it only has
to be non-empty, and `us-east-1` is the safe filler.

**The bug.** The API falls back to `us-east-1` only when the variable is completely unset
(`os.getenv("AGENTA_STORE_REGION", "us-east-1")`, `api/oss/src/utils/env.py:1093`). But every
OSS `gh` compose file injects it as an empty string (`AGENTA_STORE_REGION=${AGENTA_STORE_REGION:-}`,
`docker-compose.gh.yml:64` and `:265`, plus `gh.ssl.yml:261` and `gh.local.yml:242`). An empty
string counts as "set," so the fallback never fires, the S3 client gets an empty region, and it
refuses every request (`MissingRegion`). Helm is not affected; its template uses
`default "us-east-1"`, which treats empty as unset (`_helpers.tpl:870`).

**The fix (do both, defense in depth):**
- Code (`api/oss/src/utils/env.py:1093`): change to
  `region: str = os.getenv("AGENTA_STORE_REGION") or "us-east-1"` so an empty string is
  treated as unset. This matches the sibling fields that already use the `or` idiom.
- Compose: change the four `${AGENTA_STORE_REGION:-}` occurrences to
  `${AGENTA_STORE_REGION:-us-east-1}`.

**The change adds a fallback; it does not remove one.** Today the code falls back on unset.
After the change it also falls back on empty. A real region value flows through untouched, so
nothing that works today can break. The only inputs whose behavior changes are empty and
unset, and both are already broken.

**What to check before shipping (the cloud-safety question).** Cloud talks to real S3 or R2,
where the region is not cosmetic; it must match the bucket's actual region for request signing
to succeed. Our repo cannot see cloud's secrets. Before shipping the code change, confirm in
the private cloud repo that the cloud deployment sets a non-empty `AGENTA_STORE_REGION` that
matches the real region of the cloud store bucket. If cloud already sets a correct region, this
change does nothing to cloud. If cloud currently relies on empty or unset, that is a
pre-existing latent problem, and this is the moment to set the correct region there.

**Areas:** API code + compose. Manual pre-ship check in the private cloud repo.

---

### 3. Runner replica-id churn causes "not the owner" errors on restart

**Status: still present. Confirmed. Decided by `services/runner/src/sessions/alive.ts:31-32`
and `services/runner/src/engines/sandbox_agent/session-continuity.ts:157-210`.**

**Context first: what a replica id is and why the ownership check exists.**
The runner is the service that actually runs agent code. A "replica id" is the identity of one
running runner container. It is set once when the container starts, from
`AGENTA_RUNNER_REPLICA_ID` if present, otherwise a fresh random value
(`services/runner/src/sessions/alive.ts:31-32`).

Why it matters comes down to where an agent's work physically lives. There are two sandbox
providers. With the **local** sandbox (the default), the agent's process and working directory
live inside the runner container itself. With **Daytona**, the agent runs in a separate cloud
machine. So a local session's live state exists on exactly one runner container and can only be
resumed there. If it were resumed on a different container, that container would have none of
the state and would silently start over on the wrong machine. To prevent that, the runner
records which replica owns each local session (in Redis) and refuses to resume a local session
anywhere else. The refusal is deliberate and loud: "local sandbox requires a single runner:
replica X is not the owner of session Y ... Refusing to cold-start on the wrong host"
(`session-continuity.ts:157-177`). Daytona sessions do not have this constraint, because any
runner can reconnect to the same remote machine by its stored id
(`sandbox-reconnect.ts:1-38`).

**The bug.** The owner record in Redis has a two-minute time-to-live
(`AGENTA_SESSIONS_REDIS_OWNER_TTL_SECONDS`, default 120, `api/oss/src/utils/env.py:1298`). When
the runner restarts with no `AGENTA_RUNNER_REPLICA_ID` set, it picks a new random id. The old
owner id is still in Redis for up to two minutes, so the restarted container does not recognize
its own sessions and refuses them until the record expires. No OSS compose file sets the
variable, so self-hosters get the random path. This is exactly what the user saw.

**The single-runner fix (compose only), and why it is safe.** OSS compose runs exactly one
runner container. If we give that container a stable id, a restart keeps the same id, the owner
record still matches, and sessions resume immediately with no two-minute gap. A fixed literal
is safe here precisely because there is only one container, so "every session is owned by
`local-runner-1`" is always true. Add to the runner service environment in the OSS `gh` compose
files:
```yaml
AGENTA_RUNNER_REPLICA_ID: ${AGENTA_RUNNER_REPLICA_ID:-local-runner-1}
```
The `${...:-local-runner-1}` form still lets an operator override it. This matches the user's
own fix.

**The multiple-replicas question (the honest answer).** If a self-hoster wants to run more than
one runner container with the local sandbox, a stable id does not solve their problem, and
neither would a distinct id per container. The reason is that there is no "sticky routing" in the system. When the platform
calls the runner, it uses a single address and lets the load balancer pick any container
(`services/oss/src/agent/config.py:51-54`). Nothing routes a session back to the specific
container that owns it. The owner record is only used to detect a wrong-container attempt and
fail loudly; it is never used to steer the request to the right container. So with several
local-sandbox runners behind one address, the load balancer will regularly send a session's
next turn to a container that does not own it, and the runner will correctly refuse it.

The honest conclusion: **the local sandbox is single-runner only, by design. Scaling to
multiple runner containers requires Daytona**, whose sessions carry no container-local state
and can be resumed by any runner. This also matches the concept docs, which say "a deployment
has one logical runner" (`concepts/02-how-agents-run.mdx:22`).

**Two related things worth flagging:**

1. **The Kubernetes chart has the same restart problem, even with a single runner.** Helm does
   not set `AGENTA_RUNNER_REPLICA_ID` at all, and it uses a Deployment, so a pod restart also
   gets a new id and refuses local sessions for up to two minutes. The compose fix does not
   touch Helm. See "Decisions for you" for whether we fix Helm now or document the limitation.
2. **The configuration docs describe this variable wrongly.** The reference page calls
   `AGENTA_RUNNER_REPLICA_ID` a "replica identity for logs"
   (`docs/docs/self-host/reference/01-configuration.mdx:201`). It is not a log label; it drives
   local session ownership, and when it churns it breaks session resume. We should correct that
   line and add a short note that the local sandbox is single-runner only and that scaling
   needs Daytona.

**Areas:** compose (OSS `gh` variants) + docs. Possible Helm change, pending your decision.

---

### 4. Claude subscription login dies about once a day

**Status: still present. It is a docs-and-compose gap, fixable with no image rebuild. Decided by
`docs/docs/self-host/agents/01-use-your-own-subscription.mdx:48-75` and
`services/runner/docker/Dockerfile.gh:81`.**

**Context first: what is shared, and why the copy dies.** To run agents on a personal Claude
subscription instead of an API key, the self-hoster gives the runner container access to their
Claude Code login. That login is a token that rotates: every time Claude Code refreshes, the
provider issues a new token and retires the old one.

The key fact is that sharing the real login file is fine, but copying it is not. Two programs that
read and write the same login file stay in sync. When one refreshes, it writes the new token to
that one file, and the other reads it from the same file next time. This is exactly why you can run
two Claude Code sessions on one machine and nothing breaks. A read-write bind mount of the real
login folder is just "the same file," so the desktop and the container coordinate the same way.

A copy is different. Copying the login makes a second, separate file. When the desktop later
refreshes, it writes the new token to the original file, and the provider retires the old one. The
container's copy still holds the old token and never sees the new one, because it is a different
file. So the copy drifts out of sync and dies, usually within a day. This is the failure the user
hit, and it only happens with a copy.

**Why anyone copies instead of sharing: the user-id problem.** On Linux, every account has a number
called a user id, or uid. The first human account created on a machine gets 1000. The runner
container runs as an account that is uid 1000 (`services/runner/docker/Dockerfile.gh:81`). The
Claude login file is private, readable only by its owner, and a bind mount keeps the file's
original numeric owner. So the container can read your mounted login only when your host account is
also uid 1000. On a plain single-user Linux box it usually is, and sharing the real file just
works. When your host account is not uid 1000 (you are a second user on the machine, or you run as
root or a service account, or you use certain cloud images), the container cannot read the private
file, so today's docs tell you to copy it and re-own the copy to uid 1000. That copy is the thing
that dies.

**The fix: let every user share the real login, with no rebuild.** For a self-hoster whose uid is
not 1000, run the container as their own user at startup. This is a runtime override in the compose
file, so it needs no image rebuild. Two lines, added to the same runner override where the login
mount already goes:

```yaml
services:
  runner:
    user: "1001:1001"      # your own uid:gid, from `id -u` and `id -g`
    environment:
      HOME: /tmp
      CLAUDE_CONFIG_DIR: /agenta/harness/claude
    volumes:
      - ~/.claude:/agenta/harness/claude:rw
```

The `HOME: /tmp` line is needed because Docker sets the home directory to an unwritable path when
you run as a non-default user, and the agent writes working files under its home directory.
Pointing home at `/tmp` fixes that.

This was tested against the real runner image, not reasoned about. As uid 1000 with a login owned
by a different user, the container is denied. With the override, the container reads the login,
rewrites it on a token refresh, and the files it creates come out owned by the host user, so the
desktop keeps full access. The runner booted as the foreign user and returned a healthy status. The
one baked folder the container cannot write as a foreign user (`/pi-agent`) does not matter, because
subscription runs write into the mounted login instead, and the image was already built to run as a
foreign user.

We considered and rejected two alternatives: changing the login file's group reintroduces the drift
problem in reverse (files the container creates become unwritable by the desktop), and a
file-access-control-list trick silently breaks when the login file is set back to private.

**Recommendation.** Replace the copy method in the docs. Tell users: mount your real login folder
read-write. If your uid is 1000, that is all you need. If it is not, add the `user:` and `HOME`
lines above. No copy, no separate login, no rebuild. Correct the doc's "never log in again by hand"
claim, which is only true once you are sharing the real file rather than a copy.

**Fix (docs + skill):**
- `docs/docs/self-host/agents/01-use-your-own-subscription.mdx`: replace the copy method with the
  `user:`/`HOME` override for non-1000 users; keep the direct read-write mount for uid 1000;
  correct the "never log in again" claim.
- Self-host skill `resources/troubleshoot.md`: add an entry for the daily authentication failure
  that points at sharing the real file plus the uid override (the existing entry only covers the
  first-time permission error, not the daily death).

**One step to confirm in live QA.** The Docker test proved the container reads and rewrites the
mounted login as a foreign user, but it could not run a full agent with a real Claude login, which
needs a live subscription. The only unproven step is the harness downloading its own CLI as a
foreign user, and that step writes only to the home directory and `/tmp`, both of which tested as
writable. We should confirm it once during the live QA pass before calling this done.

**Areas:** docs + skill. Compose override is user-side; no image or repo compose change required.

---

### 5. Self-host skill forces a heavy end-to-end test instead of offering levels

**Status: still present. Confirmed in `agenta-skills/skills/self-host-agenta/resources/test.md`.**

The skill's `test.md` currently makes a full agent run mandatory: create an agent, run a prompt
and a tool call, write a session file, run again, and confirm the file persists. That is heavy
for a first check that the stack is up.

**Fix (skill): make testing tiered, and let the user choose the depth.**
- **Default, always run: a quick sanity check.** Containers healthy, API returns 200, runner
  health reachable, sign-up reaches the studio. This is fast and proves the stack is alive.
- **Then ask the user whether they want to go further,** and give the time cost so they can
  decide:
  - A short functional check: run one prompt through an agent. A few minutes.
  - A full end-to-end check of all features: agent run, tool call, and object-store persistence
    across runs. Roughly 15 to 20 minutes (we should measure the real number and put it in).
- Update `SKILL.md` (workflow step 4 and the resources note) so it describes the tiered flow
  and the prompt, not a single mandatory end-to-end test.

**Areas:** skill only (`agenta-skills` repo).

---

### 6. Agents cannot reach GitHub without Composio, and nothing says so

**Status: by design, not a bug. Confirmed.** Two findings land on the same reality:

- `discover_tools` and `discover_triggers` return 404 on a bare OSS self-host. The endpoints
  exist and are always mounted, but they route through the Composio gateway, which only
  activates when `COMPOSIO_API_KEY` is set. With no key, the provider is missing and the call
  404s. `list_schedules` worked because it reads straight from the database and never touches
  Composio (`api/entrypoints/routers.py:756-763`, `api/oss/src/core/tools/service.py:546`).
- Custom code tools (shelling out to `gh` or `curl`) are refused by the runner on purpose. The
  executor was removed for security (`services/runner/src/engines/sandbox_agent/run-plan.ts:389-391`).

Together this means a bare OSS deployment with no Composio key has neither gateway tools nor a
code escape hatch, so an agent has no way to reach an outside service like GitHub. That is a
legitimate setup, but nothing tells the self-hoster what to expect.

**Fix (docs + skill).** Add a self-hosting section that explains, in plain terms, that gateway
tool and trigger discovery need Composio (`COMPOSIO_API_KEY`), that code tools are disabled by
design, and what works without Composio. Be specific that the triggers this affects are the
event-and-app triggers (a third-party app firing your agent), not the built-in schedule
triggers, which work without Composio. Improve the self-host skill the same way, so it sets the
same expectation during setup (see item 10 for the skill's setup prompt).

**Areas:** docs + skill.

---

### 7. Mounting a host folder for your agent

**Status: docs already cover it. Add one small, end-of-setup pointer in the skill.**

The full how-to already exists in the docs:
`docs/docs/self-host/agents/04-customize-the-agent-runtime.mdx` has an "Add folders" section with
the exact volume-mount recipe for the runner, the read-write versus read-only guidance, and the
Daytona alternative. No docs change is needed here.

**Fix (skill only, kept minimal on purpose).** In the self-host skill, after setup is complete
and only when the user chose the local sandbox (not Daytona), add one short note: if you mount a
local folder into the runner, your agents can work on it, and point to the "Add folders" doc for
how. Keep it to a sentence or two. Do not add it during setup or for Daytona users, so we do not
overwhelm anyone.

**Areas:** skill only.

---

## Part 2: Already fixed in code; only image currency remains

These two were real bugs when they were reported, but the fixes have since landed. For a
self-hoster they now reduce to running a runner image built after roughly 2026-07-18, which is
another reason publishing a current runner image (item 1) matters.

### 8. Per-turn model injection cancelled pending approval prompts

**Status: fixed.** The runner sets the model once through a structured call at session start
(`services/runner/src/engines/sandbox_agent/model.ts:108`), not by injecting a `/model` slash
command every turn. Warm and approval-resume paths never re-apply the model, so a pending
approval prompt is not cancelled. This was resolved by the approval-parking work (PR #5382). No
action needed for the injection itself.

### 9. Thinking not working with fable and a Claude subscription

**Status: very likely fixed. Needs one reproduction to confirm.** Extended-thinking display is
force-enabled for every Claude run with no gating by credential type
(`services/runner/src/engines/sandbox_agent/environment.ts:913-914`). The fix landed in commit
`afff099c4c` on 2026-07-18, which is after the report. Fable is a Claude model, so it runs
through the same path.

**Action:** reproduce on a current runner image with harness `claude`, model `claude-fable-5`,
and a self-managed connection, using the `agent-release-gate` skill, and confirm thinking frames
appear on the stream.

---

## Part 3: Separate bugs for other teams (not self-hosting)

### 10. Missing-provider case returns a misleading 404, plus a skill setup prompt

When Composio is not configured, `discover_tools` and `discover_triggers` return a bare 404,
which reads as "endpoint missing" rather than "provider not configured." A clearer 501 or 503
with a "Composio not configured, set `COMPOSIO_API_KEY`" body would set expectations.

**Actions:**
- File a GitHub issue for the misleading 404, owned by the API team.
- Improve the self-host skill so setup asks the user about Composio directly: gateway tools and
  event-and-app triggers need a Composio account (a free tier exists), so the skill should ask
  "do you want to set up Composio now for app tools and triggers, or continue without it?" Be
  explicit that this is about event-and-app triggers, not the built-in schedule triggers, which
  work without Composio. This pairs with the docs in item 6.

---

## Decisions (resolved)

1. **Cloud store region (item 2): resolved.** The cloud environment variables were checked and the
   region is set correctly, so PR B is unblocked.

2. **Kubernetes runner restart (item 3): resolved, patch it.** We make the single-replica chart set
   a stable id, so a pod restart does not refuse local sessions. Multi-replica installs stay on the
   random-per-pod path, which is correct for them. A Deployment pod gets a new name on restart, so
   the fix is a fixed id gated on the replica count being 1, not a pod-name reference.

3. **Claude login (item 4): resolved, proceed.** Share the real login read-write; non-1000 users
   add the `user:` and `HOME` override. Confirm the one unproven step (harness CLI download as a
   foreign user) during the live QA pass.

---

## Suggested PR grouping

- **PR A (agenta_cloud, private): DONE, agenta_cloud #1658.** Added the runner to both matrices in
  `43-release-to-ghcr.yml`. Item 1.
- **PR B (public, code + compose): DONE, [#5404](https://github.com/Agenta-AI/agenta/pull/5404).**
  Empty-region fix (API + OSS/EE compose) and stable replica id (compose + single-replica Helm).
  Items 2 and 3.
- **PR C (public, docs): DONE, [#5406](https://github.com/Agenta-AI/agenta/pull/5406).**
  Quick-start troubleshooting for the image 404, FAQ version-pin list,
  the Composio-expectations section, the Claude login rewrite (share the real login, add the
  `user:`/`HOME` override for non-1000 users, drop the copy method) and the claim fix, the
  replica-id reference correction and single-runner note, and the optional mount pointer. Items
  1, 3, 4, 6, 7.
- **PR D (agenta-skills): DONE, agenta-skills #21.** Tiered-testing rewrite of `test.md` and
  `SKILL.md`, the Composio setup decision, the local-only folder-mount pointer, the uid-override
  login fix, and the two new troubleshoot entries (image 404 and daily token death). Items 4, 5, 6,
  7, 10.
- **Follow-up:** reproduce item 9 in live QA on a current runner image. Item 10 issue filed:
  [#5407](https://github.com/Agenta-AI/agenta/issues/5407).

Do not merge anything into main. Each PR stops at green and ready-for-review.
