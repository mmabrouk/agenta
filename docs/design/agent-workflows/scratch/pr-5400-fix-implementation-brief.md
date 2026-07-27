# PR #5400 fix implementation brief — verified against the current head

> Prepared 2026-07-20 by Claude (Fable 5). This brief re-verifies the review report at
> `pr-5400-drive-mounts-review-report.md` (same folder) against the CURRENT head of the
> PR branch and is the working document for implementation agents. The review was done
> at commit `3ac6af25160a4b2546c682137f9d3ed5865fabe0`; the branch has since gained two
> commits. **Current head: `da040e1f53fae8e31d1ff1a5ae249b917fc1362a`** (confirmed both
> via `git ls-remote`-equivalent fetch and via `gh pr view 5400`; the reviewed SHA is a
> confirmed ancestor). All line numbers in this brief are re-anchored to `da040e1f53`,
> read via `git show origin/fe-refactor/drive-surfaces:<path>` — never the working tree.
> Where this brief and the review report disagree on a line number, this brief wins;
> for the full analysis behind each item, the review report's findings sections win.
>
> Standing constraints (from the review's Execution handoff, still in force): the branch
> belongs to Arda Erzin — never push to it directly; the local working copy is a
> GitButler workspace with uncommitted overlapping changes — never read PR state from
> the working tree, never checkout/stash/reset there; merging is always Mahmoud's
> action; very large files are out of scope for "download all"; the `git_aware`
> semantics product question is still open and blocks nothing here.

## What changed since the review

Two commits landed between the reviewed SHA and the current head:

1. **`84adb07836` — "fix(drive): address review — validate mount_id, sanitize download
   headers, surface flat-page errors."** This commit responds directly to the four open
   CodeRabbit review threads (all posted 2026-07-20T08:30):
   - CodeRabbit on `models.py:45` (type `mount_id` as UUID) → fixed:
     `ArchiveMount.mount_id` is now `UUID` and the manual `UUID(...)` cast in
     `router.py` is gone. This fully resolves **review item 5**.
   - CodeRabbit on `utils.py:138` (sanitize the archive `filename` in
     `Content-Disposition`) → addressed with a new `_content_disposition_attachment`
     helper in `utils.py`, used by both the single-file download and the archive
     stream. This resolves the header-injection half of **item 10's** filename concern
     but NOT the non-latin-1 500 (verified below, see item 10 and "New concerns").
   - CodeRabbit on `service.py:903` (loop variable `sub` shadows `re.sub`, F402) →
     fixed by renaming to `sub_key`. Cosmetic; no review item.
   - CodeRabbit on `driveFlatFiles.ts:172` (mid-list page-load errors silent) →
     addressed on the frontend (`DriveExplorer.tsx`, `chatFileRefs.tsx`,
     `driveFlatFiles.ts`). Frontend scope; no review item.
2. **`da040e1f53` — "refactor(drive): move flat-file paging onto the Fern client +
   useInfiniteQuery."** Frontend-only. Rewrites
   `web/oss/src/components/Drives/driveFlatFiles.ts` to call the paged endpoint
   through a new `queryMountFilePage` wrapper in
   `web/packages/agenta-entities/src/session/api/api.ts`, backed by hand-added methods
   (`getMountFilesPage`, `archiveMountFiles`) and types in the **generated** client
   directory `web/packages/agenta-api-client/src/generated/`. This partially addresses
   follow-up ticket 5 (move `driveFlatFiles` off raw axios), but deepens the
   generated-directory drift: the "generated" client now contains even more hand-written
   code that no Fern regeneration produced, and the new `archiveMountFiles` client
   method is dead code (nothing calls it — `driveMedia.ts` still uses raw
   `fetch`/axios for the archive).

The only backend (`api/`) changes since the review are the four files in `84adb07836`:
`models.py` (UUID type), `router.py` (drop the cast), `utils.py` (the
Content-Disposition helper), and `service.py` (the `sub` → `sub_key` rename plus its
reflow). Nothing else in the review's backend findings was touched. The API-side diff
since the review is small enough to read in one sitting:
`git diff 3ac6af25..origin/fe-refactor/drive-surfaces -- api/`.

Net effect on the review's 13 pre-merge items: **item 5 is ALREADY FIXED, item 10 is
PARTIALLY FIXED, the other 11 items are all STILL NEEDED.** Because the backend diff is
tiny, most anchors moved by only 0-3 lines; every item below quotes the current code
with current line numbers.

## Per-item status

All paths are under `api/oss/src/` unless noted. "service.py" =
`api/oss/src/core/mounts/service.py` (1347 lines at head), "router.py" =
`api/oss/src/apis/fastapi/mounts/router.py` (695 lines), "utils.py" =
`api/oss/src/apis/fastapi/mounts/utils.py` (169 lines), "models.py" =
`api/oss/src/apis/fastapi/mounts/models.py` (123 lines).

### Item 1 — U+FFFF cursor-jump infinite loop — STILL NEEDED (must fix)

The sentinel append is still present, now at **service.py:1152** (was :1149). Current
code, `list_files_page`'s paging loop (service.py:1126-1167):

```python
1126        while len(kept) < limit:
1127            objs, has_more = await self.mounts_store.list_objects_page(
1128                bucket=bucket,
1129                prefix=list_prefix,
1130                start_after=start_after,
1131                max_keys=store_page,
1132            )
1133            if not objs:
1134                break
1135            jumped = False
...
1147                    if specs and not include_gitignored:
1148                        ignored_dir = _first_ignored_ancestor_dir(rel, specs)
1149                        if ignored_dir is not None:
1150                            # Skip the whole ignored subtree in one hop: U+FFFF sorts after every
1151                            # real key under `dir/`, so the next page resumes just past it.
1152                            start_after = f"{mount_base}{ignored_dir}/" + "￿"
1153                            jumped = True
1154                            break
```

The loop continues at service.py:1157-1167 (`kept` append, `next_cursor` on a full
page, `if jumped: continue`, `if not has_more: break`). The bug is unchanged from the
review: U+FFFF encodes as `EF BF BF` in UTF-8, every astral code point encodes from
`F0` and therefore sorts after the sentinel, so a key like
`node_modules/𠀀file.txt` inside a gitignored directory is re-listed forever — the
loop resets `start_after` backwards to the same sentinel on every iteration with no
exit condition.

**Change to make:** replace line 1152 with the prefix-successor jump — increment the
last byte of the prefix instead of appending a sentinel:
`start_after = f"{mount_base}{ignored_dir}" + chr(ord("/") + 1)` (i.e. `dir/` →
`dir0`), which sorts after every key under `dir/` in byte order with no exceptions.
Additionally add a forward-progress guard to the `while` loop: remember the previous
`start_after` at the top of each iteration and break (or raise) if it did not strictly
increase. Update the comment at 1150-1151 to state the byte-order rationale. Item 3's
regression test pins this fix; implement them together.

### Item 2 — archive validates after the 200 headers — STILL NEEDED (must fix)

Unchanged in structure. The handler (router.py:662-677) returns the
`StreamingResponse` immediately; every mount resolution happens lazily inside the
generator after headers are sent, so `MountNotFound` / `MountStorageUnavailable` never
reach `@handle_mount_exceptions()` and a bad request yields HTTP 200 with a corrupt or
zero-byte zip. Current handler:

```python
662    @intercept_exceptions()
663    @handle_mount_exceptions()
664    async def archive_mount_files(
665        self,
666        request: Request,
667        *,
668        archive_request: MountArchiveRequest,
669    ):
670        await self._check(request, Permission.VIEW_MOUNTS)
671
672        return await stream_mounts_archive(
673            mounts_service=self.mounts_service,
674            project_id=UUID(request.state.project_id),
675            mounts=[(m.mount_id, m.prefix, m.path) for m in archive_request.mounts],
676            filename=archive_request.filename,
677        )
```

`stream_mounts_archive` is now at **utils.py:87-141** (shifted down ~15 lines by the
new `_content_disposition_attachment` helper); its `members()` generator defers all
work to `mounts_service.iter_archive_members`. Inside the service,
`iter_archive_members` (service.py:1184-1251) already separates the phases: it first
builds the full ordered work list — the loop at service.py:1202 (`for mount_id,
prefix, source_path in mounts:`) calls `self._resolve_mount(...)` at :1203 and lists
each mount's objects — and only then starts the bounded-concurrency byte reads.

**Change to make:** hoist the resolution/work-list phase so it runs eagerly, before
the `StreamingResponse` is constructed. The clean seam: split `iter_archive_members`
into a `build_archive_work_list(project_id, mounts) -> List[...]` coroutine (the code
currently at service.py:1200-1224) and an `iter_archive_members(work) ->
AsyncIterator` that does only the prefetch/yield phase (:1226-1251). The router (or
`stream_mounts_archive`) awaits the work-list builder first — so `MountNotFound`
propagates through the decorators as a real 404 and storage-unavailable as 503 — and
only then constructs the `StreamingResponse` over the byte-read iterator. Mid-stream
read errors remain unavoidable and stay as they are. If item 8's companion refinement
is done in the same pass, the work-list entries become the named
`MountArchiveSource` DTO instead of anonymous tuples.

### Item 3 — fake-backed unit tests for `list_files_page` — STILL NEEDED (must fix)

The test file `api/oss/tests/pytest/unit/test_mounts_file_ops.py` is byte-identical to
the reviewed SHA (empty diff, still 473 lines). The in-memory `FakeMountStorage`
(defined at line 70) still implements only `list_objects_v2` (:75),
`list_objects_shallow` (:83), `get_object` (:105), `put_object` (:111), `delete_keys`
(:115), and `delete_prefix` (:124) — **it does not implement `list_objects_page`**, so
`list_files_page` still cannot be exercised at all. No new tests were added anywhere
under `api/oss/tests/` since the review.

**Change to make:** implement `list_objects_page(*, bucket, prefix, start_after,
max_keys) -> Tuple[List[StoreObject], bool]` on `FakeMountStorage` with byte-order
(UTF-8-encoded key) sorting to match S3 semantics — the sort order is the whole point
of the item-1 bug, so sort by `key.encode("utf-8")`, not by `str`. Then add tests: (a)
plain paging across page boundaries (write more files than one `limit`, walk
`next_cursor` to exhaustion, assert the union is complete, ordered, and non-repeating);
(b) the git-aware ignored-directory jump with an astral-plane filename — a gitignored
directory containing e.g. `"\U00020000file.txt"` — asserting the pager terminates,
skips the whole ignored subtree, and returns the surrounding keys. Test (b) must fail
against the current sentinel code and pass with item 1's fix. The review's
verification command: from `api/`,
`uv run pytest oss/tests/pytest/unit/test_mounts_file_ops.py -x -q`.

### Item 4 — rename `POST /files/archive` — STILL NEEDED (recommended)

The route is unchanged at router.py:197-204, still colliding with the soft-delete
`archive_mount` registered directly below (:205-213) and still carrying the
ordering-hack comment:

```python
195        # Registered BEFORE "/{mount_id}/archive" so `POST /files/archive` (download-all zip) isn't
196        # captured as archiving a mount literally named "files".
197        self.router.add_api_route(
198            "/files/archive",
199            self.archive_mount_files,
200            methods=["POST"],
201            operation_id="archive_mount_files",
```

**Change to make:** rename the path to a non-lifecycle verb — the review suggests
`/files/export` or `/files/zip` (pick one; `/files/export` reads better next to the
existing `/files/download`). Rename the handler and `operation_id` to match (e.g.
`export_mount_files`), and update or drop the ordering comment (the registration-order
hack is still needed only if the new name can be captured by `/{mount_id}/archive`,
which it cannot — but it CAN still be captured by other `/{mount_id}/...` routes, so
keep it registered early and reword the comment). **The rename now has THREE frontend
touchpoints, one more than at review time** (commit `da040e1f53` added the third):

- `web/oss/src/components/Drives/driveMedia.ts:232` — the streaming download path:
  ``const url = `${getAgentaApiUrl()}/mounts/files/archive?project_id=${encodeURIComponent(projectId)}` ``
- `web/oss/src/components/Drives/driveMedia.ts:260` — the buffered fallback:
  ``await axios.post(`${getAgentaApiUrl()}/mounts/files/archive`, payload, ...)``
- `web/packages/agenta-api-client/src/generated/api/resources/mounts/client/Client.ts`
  — the hand-added `archiveMountFiles` method builds `"mounts/files/archive"` (around
  line 559) and names the route in its error string (around :594). This method is
  currently dead code, but it must not be left pointing at a route that no longer
  exists; update its URL (and consider renaming the method to match the new
  `operation_id`, since a future real Fern regeneration will do exactly that).

All three must change in the same PR as the route rename so the branch stays
self-consistent. `web/packages/agenta-entities/src/session/api/api.ts` does NOT call
the archive route (verified); no change needed there.

### Item 5 — `ArchiveMount.mount_id` as UUID — ALREADY FIXED

Commit `84adb07836` made exactly the requested change. Current code, models.py:44:
`mount_id: UUID` (with the `from uuid import UUID` import added), and router.py:675
now passes `m.mount_id` with no manual cast. A malformed id now 422s via Pydantic.
Nothing to do. When the fixes land, resolve the corresponding CodeRabbit thread on
`models.py:45` if it has not auto-resolved.

### Item 6 — constrain `depth` — STILL NEEDED (recommended)

Unchanged. The router advertises `ge=1` at **router.py:515**
(`depth: Optional[int] = Query(default=None, ge=1)`) but the service only implements
depth 1: the shallow branch fires only on `if depth == 1 and order is None and limit
is None:` at **service.py:845**, so `depth=2` silently falls through to the full-tree
browse listing — the most expensive branch.

**Change to make:** change the router constraint to `ge=1, le=1` (or, equivalently,
reject any value other than 1 with a 422) so unsupported depths fail loudly instead of
silently selecting the full enumeration. One-line change at router.py:515 plus a word
in the handler docstring.

### Item 7 — make `order` a Literal/enum — STILL NEEDED (recommended)

Unchanged. **router.py:513**: `order: Optional[str] = Query(default=None)` accepts any
string; the service dispatch at **service.py:996-1014** compares against `"recent"`
(:996), `"name"` (:1008), `"path"` (:1010), and any other value falls through all
three and returns the flat view unsorted with no error (the final `return` at :1014).

**Change to make:** type the router parameter as
`Optional[Literal["recent", "name", "path"]]` so anything else 422s at the boundary.
The service signature can adopt the same Literal for documentation value, but the
router constraint is the load-bearing change.

### Item 8 — named DTO for `list_files_page`'s return — STILL NEEDED (recommended)

Unchanged. **service.py:1063-1073**: `list_files_page` is declared
`-> Tuple[List[MountFile], Optional[str]]` (the annotation on :1073), the exact bare
tuple shape `api/AGENTS.md` names as an anti-pattern. It is unpacked at
**router.py:569**:

```python
569        files, next_cursor = await self.mounts_service.list_files_page(
...
578        return MountFilePageResponse(
579            count=len(files),
580            files=files,
581            next_cursor=next_cursor,
582        )
```

**Change to make:** add a `MountFilePage` DTO (fields `files: List[MountFile]`,
`next_cursor: Optional[str]`) in `core/mounts/dtos.py` next to its siblings, return it
from the service, and read `.files` / `.next_cursor` in the router. Per the Codex
refinement adopted in the review, do the archive path's tuples in the same pass: the
`(UUID, str, str)` triples that travel router → `stream_mounts_archive` →
`iter_archive_members` (router.py:675, utils.py:92 `mounts: List[Tuple[UUID, str,
str]]`, service.py:1188) become a named
`MountArchiveSource(mount_id, archive_prefix, source_path)` DTO. Note the mild overlap
with item 1 (same function body) and item 2 (same archive signatures): implement
after those or in the same PR, not in a parallel one.

### Item 9 — unit tests for `_rollup_recent_entries` — STILL NEEDED (recommended)

Unchanged. `_rollup_recent_entries` sits at **service.py:229-329** (was :229-327;
shifted 2 lines by the `sub_key` reflow), still a pure
`List[MountFile] → List[MountFile]` function with zero tests. The behaviors to pin are
listed in the review's Finding 3.3: clone-then-edit collapses to the top directory; an
old-plus-fresh directory does not collapse; an untimed leaf blocks collapse; a
single-batch mount produces no rollup (pin the current behavior — the fallback is a
product decision deferred to follow-up ticket 7); shallow-to-deep resolution picks
`repo/` over `repo/web/`; and the file-and-directory-same-key edge (the file row is
silently dropped, currently at service.py:322). Pure-function tests, no fake store
needed; add a new test class in `test_mounts_file_ops.py`.

### Item 10 — validate the archive body's path-shaped inputs — PARTIALLY FIXED

**What `84adb07836` fixed:** the `filename` header-injection vector. Both
`download_mount_file` (utils.py:83) and `stream_mounts_archive` (utils.py:137) now
route through the new helper at **utils.py:20-31**:

```python
20    def _content_disposition_attachment(filename: str) -> str:
...
28        safe = (
29            "".join(c for c in filename if c.isprintable() and c not in '"\\') or "download"
30        )
31        return f"attachment; filename=\"{safe}\"; filename*=UTF-8''{quote(filename)}"
```

Quote and control-character breakout is fixed (`isprintable()` drops controls, the
filter drops `"` and `\`).

**What remains — three gaps:**

1. **The non-latin-1 500 is NOT fixed.** `str.isprintable()` keeps every printable
   Unicode character, so a filename like `中文 report.zip` survives into the plain
   `filename="..."` parameter; Starlette encodes response headers as latin-1 and
   raises `UnicodeEncodeError`, which is still a 500 (verified by executing the
   filter and the latin-1 encode). The docstring's "ASCII-ish fallback" claim is
   wrong. Fix: tighten the fallback filter to
   `c.isascii() and c.isprintable() and c not in '"\\'` — the `filename*` parameter
   already carries the exact UTF-8 name for modern clients. Also pass `safe=""` to
   `quote(filename)` so `/` gets percent-encoded in `filename*` (cosmetic but
   correct). Add a unit test asserting the returned header value encodes to latin-1
   for CJK, emoji, quote, and control-character inputs.
2. **`ArchiveMount.path` still skips `validate_file_path`.** models.py:44-46 has no
   validation and neither the router (:662-677) nor `iter_archive_members` validates
   `path`/`prefix`. Fix: in the handler (or a Pydantic validator on `ArchiveMount`),
   call `validate_file_path` on each non-empty `path` and `prefix` — using the
   item-11 relaxed validator, since these are the same real-world folder names.
3. **Zip entry names are still built from raw input.** service.py:1205-1206 only
   strips slashes (`pfx = prefix.strip("/")`, `src = source_path.strip("/")`) and
   :1223 builds `zip_path = f"{pfx}/{rel}" if pfx else rel`, where `rel` comes from
   raw store keys that signed-credential writers control — so `..` segments can
   appear in zip entry names ("zip-slip" for whoever extracts). Fix: when building
   `zip_path`, drop or reject `..` and empty segments from both `pfx` and `rel`
   (a small `_safe_zip_path` helper; skipping the offending member with a log is
   acceptable since the requester mostly harms their own download).

Items 10 and 11 both touch `validate_file_path` from opposite directions (10 applies
it to new inputs, 11 relaxes what it accepts); one agent owns both together.

### Item 11 — relax `validate_file_path`'s segment allowlist — STILL NEEDED (must fix)

Unchanged. Current code:

```python
42    # Folder/file path segments: word chars, dots, spaces, hyphens — no path traversal.
43    _SEGMENT_RE = r"[\w. -]+"
...
109    def validate_file_path(path: str) -> None:
...
116        if path.startswith("/"):
117            raise MountPathInvalid("File path must not be absolute.")
118        if not path.strip("/"):
119            raise MountPathInvalid("File path must not be empty.")
120        for segment in path.split("/"):
121            if not segment:
122                continue
123            if segment == ".." or not fullmatch(_SEGMENT_RE, segment):
124                raise MountPathInvalid()
```

The allowlist still rejects `( ) [ ] @ + , # ~` and quotes, so real folders agents
create — `app/(auth)/[slug]`, `@scope/pkg`, `c++.md` — appear in listings but 422 on
every expand, page, read, or download. The PR's lazy-browse flow round-trips `path` on
every folder expansion, which is what promotes this latent restriction into a
headline-flow bug.

**Change to make:** replace the character allowlist with a denylist of the genuinely
unsafe: keep the absolute-path and empty checks, reject a segment that is `.` or `..`,
and reject NUL and control characters (`ord(c) < 0x20` or `c == "\x7f"`) anywhere in
the path; drop the `_SEGMENT_RE` allowlist (or redefine it as the control-char
denylist). Callers are unchanged — `validate_file_path` is called at service.py:828
(`list_files`), :1091 (`list_files_page`), :1178, :1275, :1293, :1314 (read, write,
folder, delete) and via the routers. Extend `TestFilePathValidation` (test file :34-67)
with realistic accepts (`app/(auth)/[slug]/page.tsx`, `@scope/pkg/index.js`, `c++.md`,
`a,b.txt`, `notes#1.txt`, `~backup`) and keeps-rejecting cases (`..` middle segment,
absolute, embedded NUL/control chars, `.` segment). Design together with item 10's
tightening (opposite directions on the same function).

### Item 12 — one meaning for `total` — STILL NEEDED (recommended)

Unchanged; anchors shifted slightly. The documented meaning lives at
**models.py:76-83** (`MountFileListResponse`, `total` at :80: "Full file count
matching the request before any limit"). The three computed meanings: shallow mode
returns immediate files-plus-folders (`return MountFileList(files=shallow,
total=len(shallow))`, **service.py:950**); recency/flat mode returns the pre-limit
leaf-file count (`total = len(files)` at **:993**, returned at :995/:1002/:1007/:1014);
browse mode returns recursive files-plus-synthesized-folders
(`return MountFileList(files=browse_files, total=len(browse_files))`, **:1061**).

**Change to make:** pick one contract. The cheapest honest fix is documentation-plus-
naming: reword the `total` docstring at models.py:78-80 to state the per-mode meaning
explicitly (files-only in flat/recency mode; entries-including-folders in
shallow/browse modes). The stronger fix is per-view fields, but that changes the wire
contract the frontend already consumes — do not do that without checking the three
frontend consumers of `total`. Either way this is a small, self-contained change; the
review requires only that the field stop having three undocumented meanings.

### Item 13 — cap the plain count-only path — STILL NEEDED (recommended)

Unchanged. Current code, service.py:953-970:

```python
953        if order is not None or limit is not None:
...
956            count_only = limit == 0 and order is None
957            cap = _COUNT_CAP if count_only else None
958            if git_aware:
...
961                store_files, specs, truncated = await self._list_pruned_files(
962                    base_prefix=list_prefix, mount_base=mount_base, cap=cap
963                )
964            else:
965                # RAW: every object under the prefix, no pruning (matches the plain-endpoint contract).
966                objects = await self.mounts_store.list_objects_v2(
967                    bucket=self._bucket(), prefix=list_prefix
968                )
969                store_files = [o for o in objects if not o.key.endswith("/")]
970                specs: List[Tuple[str, "pathspec.PathSpec"]] = []
```

The cap (`_COUNT_CAP = 20000`, service.py:154) is only passed to the git-aware pruned
descent; the plain branch runs a full uncapped `list_objects_v2` enumeration and
hardcodes `truncated = False` (the line after :970). A raw count-only request on a
huge tree enumerates everything.

**Change to make:** when `count_only` and not `git_aware`, bound the enumeration to
`_COUNT_CAP` — either via a capped paged walk using `list_objects_page` (count keys
until the cap, set `truncated=True` if more remain) or by truncating after
`list_objects_v2` if the store API cannot stop early (weaker but still honest:
`truncated = len(store_files) > _COUNT_CAP` then slice). Prefer the paged walk — the
whole point is not enumerating. Set `total_capped` accordingly so the UI's "N+"
affordance works in both modes. Add a fake-backed test (the fake gains
`list_objects_page` in item 3 anyway).

## New concerns from the new commits

1. **The Content-Disposition fallback is not latin-1-safe** (detailed under item 10).
   The new sanitizer in `84adb07836` fixes injection but still 500s on any printable
   non-latin-1 filename, which the review had explicitly called out. Fold the
   ASCII-only fix into the item 10/11 work unit.
2. **The generated-client drift got deeper.** `da040e1f53` hand-writes ~230 more lines
   into `web/packages/agenta-api-client/src/generated/` (a `getMountFilesPage` and an
   `archiveMountFiles` method plus five type files). The directory now matches no Fern
   regeneration run even more thoroughly, and `archiveMountFiles` is dead code
   (nothing calls it; `driveMedia.ts` still uses raw fetch/axios for the archive).
   This stays follow-up ticket 5 (one real Fern regeneration after the item-4 rename),
   but item 4's PR must at minimum keep the hand-added method's URL consistent.
3. Nothing else: the `sub` → `sub_key` rename is behavior-neutral (verified in the
   diff), and the frontend error-surfacing changes are outside backend scope. Only
   `84adb07836` touches `api/`; `da040e1f53` is frontend-only.

## Proposed PR split

Recommendation: **two stacked PRs on top of `fe-refactor/drive-surfaces`** (per the
product owner's delivery constraint; the branch itself is Arda's and is never pushed
directly). Two rather than one because the work divides into an uncontroversial
correctness tier that should be reviewable and mergeable fast, and a contract tier
that changes wire shapes and touches the frontend — different reviewers care about
each, and the correctness tier should not wait on contract discussion. One PR would
also work if review bandwidth is scarce (total diff is moderate), but the two-story
split keeps each review coherent.

**PR 1 — "fix(api): mounts file-ops correctness — cursor jump, eager archive
resolution, count cap, tests"** (base: `fe-refactor/drive-surfaces`; `api/` only)

- Item 1: prefix-successor cursor jump + forward-progress guard
  (service.py:1126-1167).
- Item 3: `list_objects_page` on the test fake (byte-order sorting) + paging tests +
  the astral-filename regression test that pins item 1.
- Item 2: eager mount resolution / work-list build before the `StreamingResponse`
  (router.py:662-677, utils.py:87-141, service.py:1184-1251).
- Item 13: `_COUNT_CAP` on the plain count-only path (service.py:953-970) + test.
- Item 9: `_rollup_recent_entries` pinning tests (service.py:229-329).

One story: the PR's new listing, counting, and archive paths now terminate, fail
loudly, and are tested. No wire-contract change (the eager-resolution change only
turns silent corrupt-zip 200s into the 404/503 the decorators already advertise).

**PR 2 — "fix(api): mounts file-ops contract — path validation, route rename, typed
params and DTOs"** (base: PR 1's branch; `api/` + `web/`)

- Items 11 + 10 (one work unit, opposite directions on `validate_file_path`): relax
  the segment allowlist to a control-char/traversal denylist + tests with real repo
  names; apply validation to `ArchiveMount.path`/`prefix`; sanitize zip entry names;
  make the Content-Disposition fallback ASCII-only (the new-commit gap).
- Item 4: rename `POST /files/archive` → `/files/export` including all three frontend
  touchpoints (`driveMedia.ts:232`, `driveMedia.ts:260`, the hand-added
  `archiveMountFiles` in the generated `Client.ts`).
- Item 6: `depth` constrained to `le=1` (router.py:515).
- Item 7: `order` as a `Literal["recent", "name", "path"]` (router.py:513).
- Item 8: `MountFilePage` DTO for `list_files_page` + `MountArchiveSource` for the
  archive tuples (builds on PR 1's item-2 seam, which is why this PR stacks on PR 1).
- Item 12: give `total` one documented meaning (models.py:76-83).

One story: tighten the file-ops wire contract — reject what the API cannot honor,
accept the paths real repos actually contain, and name the shapes — before the
frontend hardens against the current ambiguities. Item 5 is already fixed on the
branch and appears in neither PR.

Sequencing note inside PR 2: land item 11's relaxed validator before wiring item 10's
new validation call sites, so the archive inputs are validated against the corrected
rules from the start. After both PRs, resolve the open CodeRabbit threads that the
fixes complete (the `utils.py` filename thread is only half-answered until the
ASCII-only fallback lands).
