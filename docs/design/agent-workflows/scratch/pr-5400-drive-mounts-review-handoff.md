# Review brief: PR #5400 backend (mounts/files) — multi-perspective review

## Your task

Review the backend portion of `https://github.com/Agenta-AI/agenta/pull/5400` (branch
`fe-refactor/drive-surfaces`, base `main`) from four independent angles, listed below.
This document gives you the context and the groundwork already done so you don't have
to re-derive it, but it is a starting point, not a ceiling. Investigate further on your
own, pull the PR and the repo yourself if you have the means to, and raise anything you
find that isn't already listed here. The four lenses are deliberately meant to produce
different findings from the same code; don't collapse them into one generic pass.

Produce a written report, one section per lens, each with: concrete findings anchored to
a file and, where possible, a function or line; how confident you are in each finding;
and a recommendation (merge as-is / merge with a follow-up ticket / needs a change before
merge). Where two lenses point at the same piece of code from different angles, say so
explicitly rather than repeating the finding twice without cross-reference.

## Who wrote this and why the review matters

The PR is authored by Agenta's frontend engineer (Arda Erzin), who wrote the full
change end to end, including a substantial amount of backend code. It landed while the
CTO, who normally reviews backend architecture and conventions, is traveling. The person
requesting this review wants confidence that the backend portion fits the codebase's
established backend patterns before it merges, precisely because the author's primary
strength and day-to-day focus is frontend, not backend architecture.

## What the PR does and why it exists

Agenta lets an agent run in a session with a working directory ("mount") that the
product surfaces to the user as a file browser ("drive"). Two bugs motivated this PR:

1. **A UI freeze (tracked as #5367).** When an agent wrote a large file tree into its
   working directory (the PR's test case: an 11k-file git clone), opening that session's
   file browser froze the browser's main thread for 2.5+ seconds, and refroze on every
   session update. Root cause: the file browser rendered one DOM node per file, for the
   whole tree, on every render.
2. **Slow initial load, independent of the freeze.** The file browser fetched the
   *entire* file tree before showing anything, so opening a multi-GB mount could block
   for seconds before any UI appeared, regardless of the DOM-node problem.
3. **Duplicated, drifted implementations.** Two separate components implemented file
   browsing (the build page's `DriveDrawer` and the chat page's `FilesWindow`), and they
   had grown apart over time.

The PR's fix has a frontend half (virtualized rendering via `@tanstack/react-virtual`,
so only visible rows mount in the DOM; a single unified `DriveExplorer` component
replacing both old drawers) and a backend half that changes what the API can be asked
for: instead of only "give me everything under this mount," it now supports "give me
just the top level" and "give me the next 100 files," so the frontend never has to pull
a whole tree to show something on screen.

This brief is scoped to the **backend half**. The frontend touches
`web/oss/src/components/Drives/*`, `web/oss/src/components/AgentChatSlice/*`, and a few
`web/packages/agenta-entities` / `agenta-api-client` files; mention it only if a backend
finding has a direct frontend-contract consequence (e.g., a response shape frontend code
depends on).

## The repo's documented backend conventions (your ground truth to check the PR against)

This is quoted near-verbatim from `api/CLAUDE.md`, the standing convention document for
everything under `api/`. Treat it as the codebase's own stated intent, and check whether
the PR's backend code follows it, not as something to take on faith — if you think a
convention itself is questionable, say so too, that's in scope for the architecture
lens.

**Layering (required dependency direction):** Router → Service → DAO Interface → DAO
Implementation → DB. Core services depend on interfaces (`*DAOInterface`), not concrete
DB implementations. Concrete dependencies are wired only in `api/entrypoints/*`.

**Domain-level exceptions:** define domain exceptions in the core layer
(`core/{domain}/types.py` or `dtos.py`), never raise `HTTPException` from a service or
DAO. Catch domain exceptions at the API boundary (router or a decorator) and convert to
HTTP responses. Use a base exception per domain.

**Typed DTO returns:** service methods return Pydantic `BaseModel` DTOs, not raw dicts,
tuples, or `Any`. `Optional[DTO]` for a possibly-missing entity, `List[DTO]` for
collections.

**Endpoint design conventions:**
- `POST /query` for filtering/search with payload support.
- `POST /{id}/archive` and `POST /{id}/unarchive` for lifecycle transitions.
- Revisioned resources expose `/revisions/retrieve`, `/revisions/commit`,
  `/revisions/log`.
- Response envelopes include `count` plus payload (`item`/`items`).
- Explicit `operation_id` on every route.
- **Use cursor pagination via `Windowing`, not page-number pagination** (this is the
  one line most directly relevant to this PR — see the pagination question under lens 3
  below).
- Parse query params via `Depends(...)`, optionally parse a request body, and merge the
  two into one query object in `utils.py`.

**Router style:** register routes in the router class's `__init__` via
`self.router.add_api_route(...)`; use `@intercept_exceptions()` at the boundary;
keyword-only function parameters with `*`.

**Access control:** RBAC is an OSS feature, always on in both editions. Handlers call
the access-check function unconditionally; there's no edition gate around the check
itself. The vocabulary and enforcement service live in `api/oss/src/core/access/`.

Reference examples the convention doc points to: `api/oss/src/apis/fastapi/workflows/`,
`api/oss/src/core/workflows/`, `api/oss/src/dbs/postgres/workflows/`.

## What the PR actually changed (backend)

Files touched, with the PR's own added/deleted line counts:

| File | +/- |
|---|---|
| `api/oss/src/apis/fastapi/mounts/models.py` | +31/-0 |
| `api/oss/src/apis/fastapi/mounts/router.py` | +89/-0 |
| `api/oss/src/apis/fastapi/mounts/utils.py` | +62/-1 |
| `api/oss/src/core/mounts/dtos.py` | +10/-0 |
| `api/oss/src/core/mounts/service.py` | +775/-11 |
| `api/oss/src/core/store/storage.py` | +57/-0 |
| `api/oss/tests/pytest/unit/test_mounts_file_ops.py` | +214/-0 (new file) |
| `api/pyproject.toml` | +2/-0 (two new dependencies, see below) |
| `api/uv.lock` | lockfile regen, no manual review needed |

Two new runtime dependencies were added to `api/pyproject.toml`:
`pathspec>=0.12,<0.13` (gitignore-pattern matching) and `stream-zip>=0.0.83,<0.1`
(streaming zip generation). Neither library appeared in this codebase before this PR.

### The shape of the change

`MountsService.list_files` (the one existing method the whole thing hangs off) grows
from a single unconditional "list everything under this prefix" implementation into
**three selectable views**, chosen by which query parameters are set:

- **Shallow / `depth=1`** (`service.py:869`, the branch guarded by
  `if depth == 1 and order is None and limit is None:`) — one directory level only (via
  the new `ObjectStore.list_objects_shallow`, `storage.py`), optionally with
  `with_counts=true` to also report each subfolder's immediate child count via a bounded
  concurrent fan-out. This is what backs the lazy tree: expanding a folder in the UI now
  calls this instead of fetching the whole subtree.
- **Recency / flat** (`order` or `limit` set, same function, the branch starting
  `if order is not None or limit is not None:`) — a sorted, optionally-capped listing
  that also reports a `total` independent of any `limit`, so a small "latest 5 files"
  widget can show an accurate count without fetching everything.
- **Browse** (no params) — close to the original behavior: full tree plus synthesized
  folder entries, used only when the explorer is actually opened.

A fourth, genuinely new method, `list_files_page` (`service.py:1143`), backs the new
`GET /mounts/{id}/files/page` endpoint and implements **true cursor pagination**: it
loops `ObjectStore.list_objects_page` (`storage.py`, new), keeps exactly `limit` real
files, and returns an opaque `next_cursor`. This is what the frontend's infinite-scroll
flat view calls.

**Git-aware filtering** is new and opt-in via a `git_aware: bool = Query(default=False)`
parameter added to the existing plain-listing endpoint as well as the two new ones. When
off (the default, preserving the existing contract for any other caller), the endpoint
returns the raw object-store listing, `.git` internals and all. When on (only the
Agenta UI's own queries set it), the service:
- always drops the `.git` directory (`_is_git_plumbing`, `service.py:165`) and a set of
  runner-owned internal paths (`_is_internal_mount_path`, `service.py:135` — the
  `agents/` namespace and `.agenta-*` markers);
- reads `.gitignore` files from the object store and applies them via the new
  `pathspec` dependency, checking not just the literal path but every ancestor
  directory (`_path_gitignored`, `service.py:180`) — this exists specifically to catch
  deeply-nested descendants like pnpm's
  `node_modules/.pnpm/<pkg>/node_modules/...`, which a naive per-path match on a
  top-level `node_modules` pattern would miss;
- for the tree-descent used by the recency/flat git-aware path, skips reading any
  `.gitignore` that lives inside a directory already known to be ignored
  (`_load_gitignore_specs` / `_read_gitignore_specs`, `service.py:720`/`753`), capped at
  `_MAX_GITIGNORE_FILES = 100`;
- for the cursor-paginated endpoint specifically, when it walks into an ignored
  directory it doesn't page through it — it jumps the `start_after` cursor to
  `f"{mount_base}{ignored_dir}/" + "￿"` (`_first_ignored_ancestor_dir`,
  `service.py:222`, used inside `list_files_page`), a sentinel chosen because it sorts
  after every real key under that prefix in the store's lexicographic order.

**A directory-collapsing heuristic for the "recent files" view**
(`_rollup_recent_entries`, `service.py:237`) collapses a directory into one folder row
when its files' timestamps are "cohesive" — packed together more tightly than the gap
to the nearest file activity outside that directory on at least one side — implemented
as a single-linkage test using `bisect_left`/`bisect_right` over the distinct sorted
mtimes, no fixed time-window constant. Resolved shallow-to-deep. Purpose: a `git clone`
that stamps thousands of files at once should read as one event in "recent files," not
flood the list with arbitrary leaf filenames; a directory that mixes an old file with
new ones should still surface the new file individually.

**A streaming "download all" archive** (`iter_archive_members`, `service.py:1264`, and
`stream_mounts_archive` in `utils.py`) uses the new `stream-zip` dependency to stream a
zip member-by-member (never buffered whole in memory), with a bounded ordered prefetch
(`_ARCHIVE_READ_CONCURRENCY = 8` files read ahead at a time) and `ZIP_AUTO` picking
32-bit vs 64-bit zip encoding per file by actual size. A new endpoint,
`POST /files/archive`, lets one call zip several mounts together — the product's
"drive" is actually a fold of two separate mounts (the session's working directory and
a separate `agent-files` mount) placed under different path prefixes in one archive.
Bundled into this same piece of work is a genuine, already-fixed bug: the object
store's `LastModified` is epoch **milliseconds**, but `datetime.fromtimestamp` expects
**seconds** — passing milliseconds straight through overflowed to a year outside
`datetime`'s valid range and raised mid-stream, after the 200 response headers were
already sent, which is why the symptom was a silently 0-byte zip rather than a visible
error. Now divided by 1000.

**A route-ordering fix in `router.py`:** the two new routes,
`POST /files/archive` and `GET /{mount_id}/files/page`, are registered *before* their
more general siblings, `POST /{mount_id}/archive` and `GET /{mount_id}/files`
respectively (see `router.py` around lines 79-109 and 250-260 in the merged diff).
FastAPI matches routes in registration order, so without this, a request for
`/files/archive` would be captured by `/{mount_id}/archive` with `mount_id` parsed
(incorrectly) as the literal string `"files"`. The PR's fix is a comment plus manual
ordering, not a structural guard.

### Full backend diff, for direct reference

`api/oss/src/apis/fastapi/mounts/models.py`, `router.py`, `utils.py`,
`api/oss/src/core/mounts/dtos.py`, `api/oss/src/core/store/storage.py`:

```diff
diff --git a/api/oss/src/apis/fastapi/mounts/models.py b/api/oss/src/apis/fastapi/mounts/models.py
index 84af63abd6..46e2380430 100644
--- a/api/oss/src/apis/fastapi/mounts/models.py
+++ b/api/oss/src/apis/fastapi/mounts/models.py
@@ -36,6 +36,22 @@ class AgentMountQueryRequest(BaseModel):
     name: str = "default"
 
 
+class ArchiveMount(BaseModel):
+    """One mount to include in an archive. `path` scopes it to a folder within the mount ("" = the
+    whole mount); `prefix` places its files under `prefix/` in the zip (the folded drive layout)."""
+
+    mount_id: str
+    prefix: str = ""
+    path: str = ""
+
+
+class MountArchiveRequest(BaseModel):
+    """Zip several mounts into ONE archive (the drive folds cwd + agent-files into one tree)."""
+
+    mounts: List[ArchiveMount] = Field(default_factory=list)
+    filename: str = "files.zip"
+
+
 # ---------------------------------------------------------------------------
 # Response models
 # ---------------------------------------------------------------------------
@@ -57,8 +73,23 @@ class MountsResponse(BaseModel):
 
 
 class MountFileListResponse(BaseModel):
+    count: int = 0
+    # Full file count matching the request before any limit — lets a limited "latest N" listing
+    # still report the true total (the UI badge). Equals `count` for an unlimited listing.
+    total: int = 0
+    # `total` is a FLOOR (the count-only scan hit its cap) — the UI shows "N+". False when exact.
+    total_capped: bool = False
+    files: List[MountFile] = Field(default_factory=list)
+
+
+class MountFilePageResponse(BaseModel):
+    """One cursor PAGE of a mount's flat (recursive, path-sorted) file listing — the basis for the
+    Files drawer's infinite-scroll flat view. `next_cursor` is an opaque token for the following page;
+    None means the listing is exhausted."""
+
     count: int = 0
     files: List[MountFile] = Field(default_factory=list)
+    next_cursor: Optional[str] = None
 
 
 class MountFileContentResponse(BaseModel):
diff --git a/api/oss/src/apis/fastapi/mounts/router.py b/api/oss/src/apis/fastapi/mounts/router.py
index 76f01d1001..aa80cb384c 100644
--- a/api/oss/src/apis/fastapi/mounts/router.py
+++ b/api/oss/src/apis/fastapi/mounts/router.py
@@ -27,12 +27,14 @@ from oss.src.core.mounts.types import (
 
 from oss.src.apis.fastapi.mounts.models import (
     AgentMountQueryRequest,
+    MountArchiveRequest,
     MountCreateRequest,
     MountCredentialsResponse,
     MountEditRequest,
     MountFileContentResponse,
     MountFileDeletedResponse,
     MountFileListResponse,
+    MountFilePageResponse,
     MountFileWrittenResponse,
     MountFolderCreatedResponse,
     MountQueryRequest,
@@ -43,6 +45,7 @@ from oss.src.apis.fastapi.mounts.utils import (
     download_mount_file,
     merge_mount_query,
     sign_mount_credentials,
+    stream_mounts_archive,
     upload_mount_file,
 )
 
@@ -189,6 +192,16 @@ class MountsRouter:
             response_model_exclude_none=True,
             status_code=status.HTTP_200_OK,
         )
+        # Registered BEFORE "/{mount_id}/archive" so `POST /files/archive` (download-all zip) isn't
+        # captured as archiving a mount literally named "files".
+        self.router.add_api_route(
+            "/files/archive",
+            self.archive_mount_files,
+            methods=["POST"],
+            operation_id="archive_mount_files",
+            response_model=None,
+            status_code=status.HTTP_200_OK,
+        )
         self.router.add_api_route(
             "/{mount_id}/archive",
             self.archive_mount,
@@ -236,6 +249,16 @@ class MountsRouter:
             response_model=None,
             status_code=status.HTTP_200_OK,
         )
+        # Registered before "/{mount_id}/files" so `/files/page` isn't swallowed by the browse route.
+        self.router.add_api_route(
+            "/{mount_id}/files/page",
+            self.get_mount_files_page,
+            methods=["GET"],
+            operation_id="get_mount_files_page",
+            response_model=MountFilePageResponse,
+            response_model_exclude_none=True,
+            status_code=status.HTTP_200_OK,
+        )
         self.router.add_api_route(
             "/{mount_id}/files",
             self.get_mount_files,
@@ -487,6 +510,12 @@ class MountsRouter:
         *,
         path: Optional[str] = Query(default=None),
         read: Optional[str] = Query(default=None),
+        order: Optional[str] = Query(default=None),
+        limit: Optional[int] = Query(default=None, ge=0),
+        depth: Optional[int] = Query(default=None, ge=1),
+        with_counts: bool = Query(default=False),
+        git_aware: bool = Query(default=False),
+        include_gitignored: bool = Query(default=False),
     ):
         await self._check(request, Permission.VIEW_MOUNTS)
 
@@ -505,12 +534,53 @@ class MountsRouter:
             project_id=UUID(request.state.project_id),
             mount_id=mount_id,
             path=path,
+            order=order,
+            limit=limit,
+            depth=depth,
+            with_counts=with_counts,
+            git_aware=git_aware,
+            include_gitignored=include_gitignored,
         )
         return MountFileListResponse(
             count=len(listing.files),
+            total=listing.total,
+            total_capped=listing.total_capped,
             files=listing.files,
         )
 
+    @intercept_exceptions()
+    @handle_mount_exceptions()
+    async def get_mount_files_page(
+        self,
+        request: Request,
+        mount_id: UUID,
+        *,
+        path: Optional[str] = Query(default=None),
+        cursor: Optional[str] = Query(default=None),
+        limit: int = Query(default=100, ge=1, le=1000),
+        git_aware: bool = Query(default=False),
+        include_gitignored: bool = Query(default=False),
+    ) -> MountFilePageResponse:
+        """One cursor page of the flat (recursive, path-sorted) file listing under `path` — the Files
+        drawer's infinite-scroll flat view. Never enumerates the whole subtree, so it's fast on any
+        mount size; carry `next_cursor` back to fetch the next page."""
+        await self._check(request, Permission.VIEW_MOUNTS)
+
+        files, next_cursor = await self.mounts_service.list_files_page(
+            project_id=UUID(request.state.project_id),
+            mount_id=mount_id,
+            path=path,
+            cursor=cursor,
+            limit=limit,
+            git_aware=git_aware,
+            include_gitignored=include_gitignored,
+        )
+        return MountFilePageResponse(
+            count=len(files),
+            files=files,
+            next_cursor=next_cursor,
+        )
+
     @intercept_exceptions()
     @handle_mount_exceptions()
     async def write_mount_file(
@@ -589,6 +659,25 @@ class MountsRouter:
             path=path,
         )
 
+    @intercept_exceptions()
+    @handle_mount_exceptions()
+    async def archive_mount_files(
+        self,
+        request: Request,
+        *,
+        archive_request: MountArchiveRequest,
+    ):
+        await self._check(request, Permission.VIEW_MOUNTS)
+
+        return await stream_mounts_archive(
+            mounts_service=self.mounts_service,
+            project_id=UUID(request.state.project_id),
+            mounts=[
+                (UUID(m.mount_id), m.prefix, m.path) for m in archive_request.mounts
+            ],
+            filename=archive_request.filename,
+        )
+
     @intercept_exceptions()
     @handle_mount_exceptions()
     async def delete_mount_file(
diff --git a/api/oss/src/apis/fastapi/mounts/utils.py b/api/oss/src/apis/fastapi/mounts/utils.py
index 29b09e1a4f..f644b8a6af 100644
--- a/api/oss/src/apis/fastapi/mounts/utils.py
+++ b/api/oss/src/apis/fastapi/mounts/utils.py
@@ -1,13 +1,20 @@
+from datetime import datetime, timezone
 from mimetypes import guess_type
 from posixpath import basename
-from typing import Optional
+from stat import S_IFREG
+from typing import List, Optional, Tuple
 from uuid import UUID
 
 from fastapi import Response, UploadFile
+from fastapi.responses import StreamingResponse
+from stream_zip import ZIP_AUTO, async_stream_zip
 
 from oss.src.core.mounts.dtos import MountCredentials, MountFileWritten, MountQuery
 from oss.src.core.mounts.service import MountsService
 
+# Regular-file mode for archive members (owner rw, group/other r).
+_ARCHIVE_FILE_MODE = S_IFREG | 0o644
+
 
 async def upload_mount_file(
     *,
@@ -62,6 +69,60 @@ async def download_mount_file(
         )
 
 
+async def stream_mounts_archive(
+    *,
+    mounts_service: MountsService,
+    project_id: UUID,
+    mounts: List[Tuple[UUID, str, str]],
+    filename: str = "files.zip",
+) -> StreamingResponse:
+    """STREAM a zip of EVERY file across the given mounts as a binary download ("download all").
+
+    The drive folds cwd + agent-files into one tree, so each ``(mount_id, prefix)`` is placed under
+    ``prefix/`` in the zip. The archive is streamed member-by-member (never buffered whole), and the
+    service prefetches file bodies with bounded concurrency. ``ZIP_AUTO`` picks zip32/zip64 per file
+    by size, so large drives and >4 GB archives are handled.
+    """
+
+    async def members():
+        async for (
+            zip_path,
+            _size,
+            mtime,
+            body,
+        ) in mounts_service.iter_archive_members(
+            project_id=project_id,
+            mounts=mounts,
+        ):
+            # `mtime` is the store's LastModified as epoch MILLISECONDS (see StoreObject.mtime);
+            # `datetime.fromtimestamp` wants SECONDS — passing ms overflows to a year out of range
+            # and RAISES mid-stream (after 200 headers are sent), truncating the zip to 0 bytes.
+            modified_at = (
+                datetime.fromtimestamp(mtime / 1000, tz=timezone.utc)
+                if mtime
+                else datetime.now(tz=timezone.utc)
+            )
+
+            async def _data(_body=body):
+                yield _body
+
+            # Size from the actual bytes (not the pre-read listing) so a file changed between list
+            # and read can't desync the zip entry; ZIP_AUTO then picks zip32/zip64 accordingly.
+            yield (
+                zip_path,
+                modified_at,
+                _ARCHIVE_FILE_MODE,
+                ZIP_AUTO(len(body)),
+                _data(),
+            )
+
+    return StreamingResponse(
+        async_stream_zip(members()),
+        media_type="application/zip",
+        headers={"Content-Disposition": f'attachment; filename="{filename}"'},
+    )
+
+
 async def sign_mount_credentials(
     *,
     mounts_service: MountsService,
diff --git a/api/oss/src/core/mounts/dtos.py b/api/oss/src/core/mounts/dtos.py
index c3d9309ab0..858e76c165 100644
--- a/api/oss/src/core/mounts/dtos.py
+++ b/api/oss/src/core/mounts/dtos.py
@@ -63,10 +63,20 @@ class MountFile(BaseModel):
     # Object-store LastModified as epoch milliseconds; None when the store omits it. Lets the UI
     # order files by recency regardless of how they were created (bash, Write tool, upload).
     mtime: Optional[int] = None
+    # Direct-child count for a folder entry — only set when the recency view rolls a whole
+    # freshly-written directory (e.g. a `git clone`) up into ONE folder row instead of flooding the
+    # "recent files" list with its leaves. None for real files.
+    item_count: Optional[int] = None
 
 
 class MountFileList(BaseModel):
     files: List[MountFile] = Field(default_factory=list)
+    # Total real files matching the request BEFORE any limit — so a limited "latest N" listing can
+    # still report the true file count (the UI badge) without shipping the whole tree.
+    total: int = 0
+    # `total` is a FLOOR, not exact — the count-only scan hit its cap on a very large tree, so the UI
+    # shows "N+". False for an exhaustive count.
+    total_capped: bool = False
 
 
 class MountFileContent(BaseModel):
diff --git a/api/oss/src/core/store/storage.py b/api/oss/src/core/store/storage.py
index 789f737ec2..8d7fa1804b 100644
--- a/api/oss/src/core/store/storage.py
+++ b/api/oss/src/core/store/storage.py
@@ -335,6 +335,63 @@ class ObjectStore:
             )
         return results
 
+    async def list_objects_page(
+        self,
+        *,
+        bucket: str,
+        prefix: str,
+        start_after: Optional[str] = None,
+        max_keys: int = 500,
+    ) -> "Tuple[List[StoreObject], bool]":
+        """One PAGE of objects under `prefix`, in the store's lexicographic (= path) order, resuming
+        strictly AFTER `start_after`. Returns `(objects, has_more)`. Streams lazily and stops after
+        `max_keys` — so it NEVER enumerates a huge subtree; this is the basis for cursor pagination
+        (the caller carries the last key forward as `start_after`). One extra element is pulled to
+        detect `has_more`, then dropped."""
+        client = self._client()
+        results: List[StoreObject] = []
+        async for obj in client.list_objects(
+            bucket, prefix=prefix, recursive=True, start_after=start_after or None
+        ):
+            if obj is None:
+                continue
+            # We already have a full page — the presence of one more object means more remain.
+            if len(results) >= max_keys:
+                return results, True
+            last_modified = getattr(obj, "last_modified", None)
+            mtime = int(last_modified.timestamp() * 1000) if last_modified else None
+            results.append(
+                StoreObject(key=obj.object_name, size=obj.size or 0, mtime=mtime)
+            )
+        return results, False
+
+    async def list_objects_shallow(
+        self,
+        *,
+        bucket: str,
+        prefix: str,
+    ) -> "tuple[List[StoreObject], List[str]]":
+        """One directory LEVEL under `prefix` (delimiter `/`): the immediate file objects, plus the
+        immediate subdirectory prefixes (full keys ending in `/`). Lets a caller descend the tree and
+        prune whole subtrees (a gitignored `node_modules`) WITHOUT enumerating their contents — the
+        `recursive=True` flat listing has no way to exclude a prefix, so a mount full of dependency
+        files must otherwise be scanned in its entirety."""
+        client = self._client()
+        files: List[StoreObject] = []
+        subdirs: List[str] = []
+        async for obj in client.list_objects(bucket, prefix=prefix, recursive=False):
+            if obj is None:
+                continue
+            name = obj.object_name
+            # A common-prefix (subdir) or an explicit empty-folder marker both end in `/`.
+            if getattr(obj, "is_dir", False) or name.endswith("/"):
+                subdirs.append(name)
+                continue
+            last_modified = getattr(obj, "last_modified", None)
+            mtime = int(last_modified.timestamp() * 1000) if last_modified else None
+            files.append(StoreObject(key=name, size=obj.size or 0, mtime=mtime))
+        return files, subdirs
+
     async def get_object(
         self,
         *,
```

`api/oss/src/core/mounts/service.py` is the largest file in the diff (+775/-11) — too
long to reproduce in full here without bloating this brief past the point of being
readable. Pull it directly: `git diff origin/main...pr-5400-drive-surfaces --
api/oss/src/core/mounts/service.py` against a checkout of the branch, or view the PR's
"Files changed" tab. The function-by-function description above, with line numbers
against the post-merge file, should let you navigate straight to the parts each lens
cares about instead of reading top to bottom.

Note also `api/oss/src/core/store/storage.py`'s pre-existing `_client()` method (not
touched by this PR, but called by every new method added to `ObjectStore`, including the
two new ones above): it constructs a fresh `Minio` client object on every call, with no
caching or connection reuse across calls. This is an existing pattern in the codebase,
not something the PR introduced, but the PR does add several new call sites that hit it
(the shallow-listing methods in particular get called once per directory level during a
tree descent) — worth an opinion on whether that existing pattern is fine to keep
building on, or whether increased call volume makes it worth revisiting now.

### What the PR's own description says about testing

Quoted directly from the PR body: "Backend: unit tests for the paged listing in
`api/oss/tests/pytest/unit/test_mounts_file_ops.py`... Frontend: type-clean (`tsc`) and
lint-clean across the Drives module... The Fern client was not regenerated for
`/files/page`. The drive module calls it via axios, the same way it reaches its other
binary/download routes. Noted as a follow-up if we want it in the generated client."

No mention of integration tests exercising the new endpoints end to end (through the
router, with a real or fake object store, asserting on the persisted access-control
behavior), and no mention of tests for the git-aware pruning behavior, the cursor
jump-past-ignored-directory logic, or the rollup heuristic specifically — only that unit
tests exist for "the paged listing." Confirm what the 214-line test file actually
covers versus what it doesn't; don't take the PR description's summary as complete.

## The four review lenses

### Lens 1 — Organizational fit: does the backend code follow the codebase's own conventions?

The PR was written by an engineer whose day-to-day depth is frontend. The question here
isn't "is the code correct" (that's lens 4) but "would someone who works in this backend
every day recognize this as backend code written the way we write backend code here."
Check the new code in `service.py`, `router.py`, `models.py`, `dtos.py`, and `utils.py`
against every convention quoted in the "documented backend conventions" section above:
layering direction, domain exception usage (does any new code raise `HTTPException`
directly, or leak a store/library exception past the service boundary?), typed DTO
returns (note that `list_files_page` returns a raw `Tuple[List[MountFile],
Optional[str]]` rather than a named DTO — is that a deviation worth flagging, or
consistent with how the rest of this domain already does things?), and the endpoint
design conventions (see lens 2 for the pagination-specific version of this question).
Also assess: does the new code fit the *style* of the surrounding file (naming,
docstring conventions, comment density — the repo's own `api/CLAUDE.md` says to keep
AI-generated comments minimal and only explain non-obvious "why," which this PR's
comments mostly attempt to do; judge whether they succeed), or does it read as a
different hand with different habits dropped into the middle of an existing file?

### Lens 2 — Endpoint organization, interface design, and testing practice

The documented convention is `POST /query` for filtering/search with a payload, and
cursor pagination via a `Windowing` utility (referenced in `api/CLAUDE.md`, implemented
elsewhere in the codebase — find it and compare). This PR instead exposes filtering and
pagination entirely through `GET` query parameters (`order`, `limit`, `depth`,
`with_counts`, `git_aware`, `include_gitignored`, `cursor`) on `GET /{mount_id}/files`
and the new `GET /{mount_id}/files/page`, with a hand-rolled opaque cursor (a
mount-relative store key, not whatever `Windowing` produces elsewhere). Decide whether
that's a defensible choice, this is pagination over an object store's own key ordering,
not a database query, so the existing `Windowing` machinery (presumably built around DB
cursor semantics) may not fit, or whether it's a case of a second, parallel pagination
convention now existing in the codebase for no strong reason, which is exactly the kind
of drift the layering rules exist to prevent. Also look at whether `POST /files/archive`
taking a list of mounts in its body is consistent with how other "operate on several
things at once" endpoints are shaped elsewhere in the API, if any exist to compare
against.

On testing: read the actual contents of `test_mounts_file_ops.py`, not just its line
count, and assess coverage against the surface area described above (three `list_files`
view branches, the new pagination method, the git-aware pruning and its directory-jump
behavior, the rollup heuristic, the archive streaming path) using whatever your view of
this codebase's normal test bar is. Recall from the PR body: the generated API client
(Fern) was **not** regenerated for the new `/files/page` endpoint, and the frontend
calls it via raw axios instead, matching how it already reaches other binary/download
routes. Decide whether that gap (a real endpoint with no generated client entry) is an
acceptable, precedented shortcut here or something that should block merge or get an
explicit follow-up ticket.

### Lens 3 — Architecture: general solution or point patch?

Several pieces of this PR are worth weighing as "well-designed general primitive" versus
"clever one-off built to solve exactly this bug." Form your own view on each, and add
any others you find:

- **The three-views-in-one-method shape of `list_files`.** Is branching on which
  optional query parameters are set the right way to express "shallow listing" vs
  "recency view" vs "browse," or would three distinct, named service methods (as
  `list_files_page` already is, separately) have been clearer and easier to test and
  reason about independently? The current function is now several hundred lines with
  three largely-independent code paths sharing one signature.
- **Where git-awareness belongs.** `git_aware` bakes gitignore-parsing and git's
  directory-based ignore semantics directly into `MountsService`, a service whose other
  job is generic object-store-backed file storage with no inherent git concept. Is
  embedding git semantics at this layer the right call, or should "a mount that happens
  to be a git working copy" be a distinct concept (a wrapping view, a decorator, a
  separate service) layered on top of a git-agnostic mounts service? Consider both
  directions: coupling now buys simplicity and one code path; separating it buys a
  mounts service that stays conceptually clean but costs an extra layer.
- **The rollup heuristic's maintainability.** `_rollup_recent_entries`'s single-linkage
  cohesion test (no fixed time-window constant, resolved shallow-to-deep) is a genuinely
  interesting piece of algorithm design. Assess it on its own terms: is the algorithm
  correct for the cases it's meant to solve, are there degenerate inputs where it does
  something surprising (e.g., files with no mtime at all, a directory with exactly two
  files, a mount where the *only* activity ever recorded is one big batch so there's no
  "outside" gap to compare against), and, separately from correctness, will another
  engineer six months from now be able to safely modify this without re-deriving the
  reasoning from scratch? A comment explaining intent is not the same as an
  implementation being easy to safely change.
- **Concurrency and cap constants.** `_LIST_CONCURRENCY = 24`, `_ARCHIVE_READ_CONCURRENCY
  = 8`, `_COUNT_CAP = 20000`, `_MAX_GITIGNORE_FILES = 100` are all hardcoded module-level
  constants with a one-line rationale each. Are these the right order of magnitude, are
  they in the right place (should any be configurable per deployment, the way other
  tunables in this codebase go through `api/oss/src/utils/env.py`, per the repo's
  environment-config convention?), and do they compose safely, e.g. does a mount that is
  both very deep (many directory levels, stressing `_LIST_CONCURRENCY`) and has many
  `.gitignore` files (stressing `_MAX_GITIGNORE_FILES`) degrade gracefully or badly?
- **Whether this generalizes as mounts grow.** The level-by-level descent
  (`_list_pruned_files`) trades "never enumerate an ignored subtree" for "wall-clock now
  tracks tree depth, walked with bounded fan-out, rather than raw object count." Is that
  the right tradeoff at the sizes this product expects mounts to reach, and does the
  bounded fan-out (`_LIST_CONCURRENCY`) still hold up if a mount has a very wide
  directory (thousands of siblings at one level) rather than a very deep one?
- **The pre-existing `_client()` pattern** (constructs a new store client per call, no
  reuse) that this PR's new methods lean on more heavily than before, flagged above.
  Judge whether this PR is the point where that existing pattern starts to matter.

### Lens 4 — Correctness and the trust boundaries this code sits on

This lens is about whether the new code behaves correctly under adversarial or
unexpected input, not just the happy path exercised by the PR's own QA checklist. Note:
frame every finding here in terms of correctness, access-control checks, tenant/project
scoping, input handling, resource use, and data exposure through responses, avoid
reaching for alarmist framing; this is a routine internal engineering review, not an
incident report. If your own research process wants to go further (reading the object
store's access-token-issuance code path, tracing how a project-scoped identity is
established elsewhere in this codebase, comparing against how other domains in this
API handle the same class of concern), do that on your own initiative; nothing here is
telling you not to look, only asking you to write findings up in plain engineering
terms.

Concrete starting points, all in the diff above:

- Both new endpoints (`archive_mount_files`, `get_mount_files_page`) call
  `await self._check(request, Permission.VIEW_MOUNTS)`, the same check the existing
  sibling endpoints use. Confirm that's the right permission for what these endpoints
  actually do (note `archive_mount_files` only *reads*, matching `VIEW_MOUNTS`, but
  produces a bulk export across potentially several mounts in one call — is a bulk-read
  operation equivalent to a single-file read from a permission-modeling standpoint, or
  should it be treated differently?).
- `MountArchiveRequest.mounts` is a caller-supplied list of `{mount_id, prefix, path}`.
  Trace `archive_mount_files` → `stream_mounts_archive` → `iter_archive_members`
  (`service.py:1264`) and confirm each `mount_id` in that list is independently resolved
  against the *same* `project_id` taken from the request context (via
  `_resolve_mount`), and that there's no path by which a caller could reference a mount
  belonging to a different project by ID. Also confirm there's no upper bound on how
  many mounts, or how much total data, one archive request can ask for.
  `list_files_page`
  is similar: `path` is user/agent-influenced (it's a directory that exists because an
  agent or a user wrote files there) and flows into a store prefix and into
  `.gitignore` object reads; confirm the existing `validate_file_path` (used elsewhere
  in this file for the `path` parameter) is actually applied everywhere a path-shaped
  input enters the new code, including inside `list_files_page` and the git-aware
  branches of `list_files`.
- The gitignore-parsing path reads file *contents* from the object store and feeds them
  into `pathspec.PathSpec.from_lines`, wrapped in a bare `except Exception` that
  silently drops a bad file (`_load_gitignore_specs`, `_read_gitignore_specs`). Confirm
  that's an appropriate failure mode (never breaking a listing because of one malformed
  file) versus a place where a very large or pathologically-crafted `.gitignore` could
  cost more than intended before the `except` catches it, the file is read into memory
  in full via `get_object` before being handed to `pathspec`, with no size check
  visible in this diff.
- Resource use / abuse resistance: `_COUNT_CAP` bounds the count-only path, and
  `list_files_page`'s `limit` is bounded to `le=1000` at the router (`router.py`,
  `get_mount_files_page`), but the plain `GET /{mount_id}/files?limit=...&order=...`
  path takes `limit: Optional[int] = Query(default=None, ge=0)` with **no upper bound**.
  Confirm whether that's intentional (this path returns a capped/count-only response in
  some branches) or a gap relative to the new paginated endpoint's explicit `le=1000`.
- Data exposure through error responses: confirm the new endpoints' error paths (a
  missing mount, an invalid cursor, a path that doesn't exist) go through the same
  domain-exception-to-HTTP translation as the rest of this domain (per the
  "domain-level exceptions" convention above) rather than leaking a raw store or
  library exception message to the client.
- The mtime bug already fixed in this same PR (milliseconds passed where seconds were
  expected, described above) is worth an independent look for siblings: are there other
  places in this diff, or in the surrounding code the diff touches, doing arithmetic on
  a timestamp or a size where a unit mismatch could produce a wrong-but-not-obviously-
  broken result rather than a hard failure?

## What's already been checked and is not an open question

- No database migration is part of this PR.
- The backend and frontend were rebuilt and deployed to a local instance; both started
  cleanly with no import errors from the two new dependencies, and a smoke pass through
  the new file-browser behavior (opening a large mount, switching views, reading files)
  worked as described in the PR.
- One single-file read was observed taking roughly 20 seconds on first access in that
  local instance. Investigated and ruled out as a PR issue: the code path involved
  (`read` query param → `MountsService.read_file` → `read_file_bytes` → a single
  `get_object` call) has zero lines changed in this PR; the file itself is 13KB; and
  three immediate repeat requests for the same file completed in 10-40 milliseconds
  each. Most likely a one-time environment effect (the API process had just been
  restarted to pick up the new dependencies) rather than anything reproducible. Treat
  this as closed unless your own testing reproduces it differently.

## Reference material

- PR: `https://github.com/Agenta-AI/agenta/pull/5400` — "[AGE-3965] refactor(drive):
  virtualize surfaces + unify Files drawer," branch `fe-refactor/drive-surfaces` into
  `main`, 8 commits, +7279/-2938 across 51 files.
- Standing backend convention document (quoted from above, read in full if useful):
  `api/CLAUDE.md` at the repository root.
- Domain folder structure to compare against, per that convention doc's own worked
  examples: `api/oss/src/apis/fastapi/workflows/`, `api/oss/src/core/workflows/`,
  `api/oss/src/dbs/postgres/workflows/`.

<!-- CODEX-REVIEW-START -->

## [Codex] Backend review report

> **[Codex] Review target:** PR #5400 at `3ac6af25160a4b2546c682137f9d3ed5865fabe0`,
> compared with `origin/main` after a fresh fetch on 2026-07-20. This report covers the
> backend diff and direct frontend contract consequences only.

### [Codex] Executive verdict

**[Codex] Recommendation: needs changes before merge.** The route permission and project
scoping are correct, the object-store cursor is a reasonable reason not to reuse the
database-oriented `Windowing.next: UUID`, and the API, integration, acceptance, format, and
lint checks are green. Those positives do not cover several untested correctness and resource
problems in the new code.

The minimum pre-merge set is:

1. Make archive generation byte-streaming and validate every archive entry path. Do not hold up
   to eight complete file bodies in memory.
2. Replace the U+FFFF ignored-directory cursor sentinel with a real prefix-successor operation,
   and add a regression test with a filename that sorts after U+FFFF.
3. Make lazy-browse path validation accept ordinary filesystem names while still rejecting
   absolute paths, traversal segments, NUL, and other genuinely unsafe input.
4. Type archive mount IDs as `UUID`, sanitize the response filename, and validate archive request
   cardinality and path fields. The first two items already have CodeRabbit comments; this report
   independently confirms them.
5. Add direct tests for `list_files_page`, archive streaming, the recent-directory rollup, nested
   gitignore overrides, and route-level request validation. The current added test code does not
   call `list_files_page`.
6. Either make `git_aware` behavior consistent with Git across views or document and name it as a
   best-effort presentation filter. The current single flag has different semantics in shallow,
   recency, browse, and page modes.

### [Codex] Information sufficiency and assumptions

**[Codex] No missing information blocks this review.** Two operational contracts are not stated:
the supported maximum single-file/archive size, and whether `git_aware=true` promises Git-compatible
ignore semantics. I assumed the archive must support multi-GB drives because the new docstrings say
that it does, and that a parameter named `git_aware` should behave consistently enough not to return
files Git would ignore or hang while skipping them. If either assumption is intentionally weaker,
the API and docs should say so explicitly.

### [Codex] Lens 1: organizational fit with backend conventions

| Codex finding | Evidence and assessment | Confidence | Recommendation |
|---|---|---|---|
| **[Codex L1-F1] The new paged service contract returns an unnamed tuple instead of a core DTO.** | `MountsService.list_files_page` returns `Tuple[List[MountFile], Optional[str]]` at `api/oss/src/core/mounts/service.py:1061`, and the router unpacks it at `api/oss/src/apis/fastapi/mounts/router.py:569`. This departs from the documented typed-DTO return convention. A `MountFilePage` core DTO would give the cursor contract one canonical home. | High | **Needs a change before merge.** |
| **[Codex L1-F2] Archive inputs lose their meaning at the core boundary.** | `ArchiveMount` is a Pydantic API model, but the router converts it into `List[Tuple[UUID, str, str]]`, which flows through `stream_mounts_archive` into `iter_archive_members` (`models.py:39`, `router.py:664`, `utils.py:72`, `service.py:1182`). The tuple slots represent source identity, source path, and archive destination mapping, which are different semantic roles. | High | **Needs a change before merge.** Use a named core DTO such as `MountArchiveSource(mount_id, source_path, archive_prefix)`. |
| **[Codex L1-F3] The code follows the router, permission, and project-scope conventions.** | Both new routes have explicit operation IDs, boundary decorators, keyword-only options, and unconditional `VIEW_MOUNTS` checks (`router.py:195`, `252`, `551`, `662`). `_resolve_mount` queries by both `project_id` and `mount_id` for every archive source (`service.py:583` and `1200`). No new service code raises `HTTPException`. | High | **Merge as-is for these aspects.** `VIEW_MOUNTS` is consistent with existing single-file reads. |
| **[Codex L1-F4] The PR deepens an existing concrete storage dependency and makes the service substantially less cohesive.** | `MountsService` already depends on concrete `ObjectStore`, so the PR did not create that debt. It adds shallow paging, Git filtering, temporal rollup, and ZIP orchestration directly to the same file, adding 775 lines. The narrative comment density also exceeds the API guide's preference for terse comments that explain only non-obvious invariants. | High on fit; medium on timing | **Merge with a follow-up ticket** for an object-store interface and service extraction, after the functional blockers are fixed. |
| **[Codex L1-F5] Predictable archive failures occur after the HTTP response can start.** | Mount resolution and object listing happen inside the `StreamingResponse` iterator (`utils.py:87` and `service.py:1198`). A missing second mount can therefore produce a truncated 200 download instead of a normal 404. New list adapter failures also become generic 500s instead of `MountStorageUnavailable`. | High | **Needs a change before merge.** Preflight mount IDs and request paths before returning the streaming response; translate adapter failures at the store boundary. |

### [Codex] Lens 2: endpoint organization, interface design, and testing practice

| Codex finding | Evidence and assessment | Confidence | Recommendation |
|---|---|---|---|
| **[Codex L2-F1] The GET verbs and native object-store cursor are defensible.** | These are read-only listing operations. Shared `Windowing` is database-shaped: its `next` token is a UUID (`sdks/python/agenta/sdk/models/shared.py:203`), while this endpoint must resume after an object key. Reusing it would misrepresent the storage contract. | High | **Merge as-is** on verb and cursor mechanism. Model the string cursor in a page DTO and document that it belongs to one query shape. |
| **[Codex L2-F2] One endpoint expresses several operations through optional fields and silently accepts invalid combinations.** | `get_mount_files` combines file reading with shallow, recency, count-only, and full-browse listing (`router.py:506`). Only exactly `depth == 1` selects shallow mode, `order` or `limit` selects flat mode, and everything else selects full browse (`service.py:845`, `951`, `1014`). Thus `depth=2` means unlimited recursive browse, `depth=1&limit=5` ignores depth, an unknown `order` returns an unsorted flat listing, and `with_counts` is ignored outside shallow mode. | High | **Needs a change before merge.** Use literal/enum fields and reject conflicting combinations with a typed query model. Prefer named listing methods or a discriminated `view`. |
| **[Codex L2-F3] The response field `total` changes meaning across modes.** | The response documents it as full file count (`models.py:75`). Recency returns leaf-file count before limit (`service.py:991`), shallow returns immediate file-plus-folder entry count (`service.py:948`), and browse returns files plus synthesized folders (`service.py:1059`). | High | **Needs a change before merge.** Use view-specific DTOs or one stable meaning plus separately named counts. |
| **[Codex L2-F4] The claimed pagination tests are absent.** | The added fake implements `list_objects_shallow` but not `list_objects_page`. Added tests call `list_files` for Git-aware, count-only, and shallow cases (`test_mounts_file_ops.py:190` through `380`); no test calls `list_files_page`, passes a cursor, or checks `next_cursor`. There are also no new router, archive, mtime ZIP, or rollup tests. | High | **Needs a change before merge.** Add direct service tests and at least one router-level test for validation, access, response shape, and project scoping. |
| **[Codex L2-F5] The new JSON endpoint is missing from the generated client.** | `driveFlatFiles.ts:1` through `82` calls `/files/page` with raw axios and manually casts the response. This is a normal JSON route, not a binary route. CodeRabbit has already raised this guideline violation. | High | **Merge with a follow-up ticket only if explicitly tracked and accepted.** Prefer regenerating the client in this PR. |

### [Codex] Lens 3: architecture, generality, and maintainability

| Codex finding | Evidence and assessment | Confidence | Recommendation |
|---|---|---|---|
| **[Codex L3-F1] `list_files` is now a dispatcher for three independent algorithms.** | The method spans `service.py:787` through `1059` and selects behavior by optional-field presence. The branches differ in store primitive, filtering, result semantics, and cost. This structure caused the silent-combination and `total` contract problems in Lens 2. | High | **Needs a change before merge** at the method boundary. Extract explicit directory, recent-files, and full-browse helpers with typed inputs/results. |
| **[Codex L3-F2] Git-aware browsing is useful product policy embedded in a generic storage service.** | Keeping filtering close to the object-store walk avoids duplicate scans, but `MountsService` now owns pathspec parsing, rule scope, internal-file policy, cursor skipping, and hidden-file presentation. The behavior already differs by listing mode. | High | **Merge with a follow-up ticket** to extract a tested Git-aware filter/listing component after current correctness gaps are fixed. |
| **[Codex L3-F3] The recent-directory rollup misses its stated motivating case.** | `_rollup_recent_entries` requires a timestamp outside the directory because `_cohesive` returns `bool(gaps)` (`service.py:229` through `327`). A new mount containing only one cloned repository has no outside timestamp, so it does not roll up. Conversely, `max(gaps)` lets one distant outside timestamp classify a directory with a wide internal history as cohesive. No tests cover either case. | High | **Needs a change before merge.** Define only-batch, mixed old/new, equal-time, missing-mtime, and distant-neighbor examples first, then simplify or replace the heuristic. |
| **[Codex L3-F4] “Latest N” bounds the response, not backend work.** | The recency branch enumerates every surviving object, builds every `MountFile`, sorts, then applies `limit` (`service.py:950` through `1012`). The frontend acknowledges this full scan at `web/packages/agenta-entities/src/session/api/api.ts:470`. Exact `total` makes a full scan unavoidable with the current storage model. | High | **Merge with a follow-up ticket** if expected repositories are modest after ignore pruning. For large mounts, index/cache counts and recent metadata. Do not describe the route as bounded work. |
| **[Codex L3-F5] The ZIP streams HTTP output but buffers complete object bodies.** | `ObjectStore.get_object` reads a whole object into `bytes` (`storage.py:395`). `iter_archive_members` builds the complete work list, starts up to eight full-body reads, and holds those byte strings in tasks (`service.py:1198` through `1244`). A few multi-GB files can exhaust API memory. ZIP64 support does not make object reads memory-safe. | High | **Needs a change before merge.** Add a streaming object-body API and feed bounded chunks to `stream-zip`. Bound concurrency by bytes, not only file count. |
| **[Codex L3-F6] The advertised Gitignore cap does not bound a level batch.** | `_read_gitignore_specs` gathers every read concurrently and has no local cap (`service.py:671`). `_list_pruned_files` checks `len(specs) < 100` and then passes the whole level batch (`service.py:741` through `766`), so one level can exceed `_MAX_GITIGNORE_FILES`. The traversal semaphore limits active I/O, not task/frontier size. | High | **Needs a change before merge.** Slice remaining reads and use bounded worker iteration. |
| **[Codex L3-F7] Recreating `Minio` clients deserves measurement, not a speculative rewrite.** | `_client()` constructs a client for every adapter call (`storage.py:107`), and the new level walk increases calls. Current evidence does not show client construction is material relative to network latency. | Medium | **Merge with a follow-up profiling ticket** only if store telemetry shows it matters. |

### [Codex] Lens 4: correctness, trust boundaries, and data handling

| Codex finding | Evidence and assessment | Confidence | Recommendation |
|---|---|---|---|
| **[Codex L4-F1] The ignored-directory cursor jump can loop forever for valid Unicode filenames.** | `list_files_page` jumps to `ignored_dir + "/" + U+FFFF` and claims it sorts after every key (`service.py:1146`). U+FFFF is not the maximum Unicode scalar. A key such as `node_modules/😀.js` sorts after that sentinel in the lexical ordering the code relies on. The next page can return the same ignored object, set the same sentinel again, and repeat without progress. Agent-mounted filenames are not restricted by `validate_file_path`. | High | **Needs a change before merge.** Compute the lexicographic successor of the ignored prefix, assert strict cursor progress, and regression-test non-BMP characters. |
| **[Codex L4-F2] Archive entry paths are caller-controlled and can contain traversal segments.** | `prefix.strip("/")` leaves `../../outside` unchanged, and `zip_path = f"{pfx}/{rel}"` writes it directly (`service.py:1200` through `1222`). `ArchiveMount.prefix` and `path` have no constraints (`models.py:39`). Duplicate destinations are also accepted. | High | **Needs a change before merge.** Normalize every final archive path as relative POSIX, reject absolute paths, `.`/`..`, backslashes/control characters, and collisions, and constrain source paths. |
| **[Codex L4-F3] The response filename and archive mount ID are insufficiently typed.** | `mount_id: str` is manually passed to `UUID(...)`, so malformed JSON becomes a generic 500 (`models.py:43` and `router.py:675`). `filename` is interpolated into `Content-Disposition` without quote/control-character handling (`utils.py:119`). CodeRabbit has open comments for both and this review confirms them. | High | **Needs a change before merge.** Use `UUID`, a constrained non-empty mount list with a maximum, and RFC 6266/5987-safe filename encoding. |
| **[Codex L4-F4] The path allowlist rejects common repository paths required by lazy browsing.** | `validate_file_path` allows only `[\w. -]+` per segment (`service.py:109`). Agent-created paths such as Next.js `app/(auth)/[slug]`, npm-style `@scope`, and names containing `+`, `,`, or `#` can be listed but cannot be passed back to expand, page, read, or download. The new lazy flow sends `path` on each expansion. | High | **Needs a change before merge.** Preserve relative paths while rejecting absolute paths, traversal segments, NUL, and actual escape conditions. Add realistic repo-name tests. |
| **[Codex L4-F5] Gitignore behavior is not Git-compatible across nested rules or page mode.** | `_path_gitignored` returns on the first matching spec (`service.py:172`), so a deeper `.gitignore` cannot override a root file pattern for a kept directory. Page mode loads only root-to-request-path files and explicitly skips deeper ignore files (`service.py:1078`). Shallow child counts do not load each child's own `.gitignore`, so counts can disagree with expanded contents. | High | **Needs a change before merge** if `git_aware` promises Git behavior. Otherwise rename/document it as best-effort and add supported/unsupported contract tests. |
| **[Codex L4-F6] Page exhaustion and cursor context are underspecified.** | On exactly `limit` entries, the method returns a cursor without considering that `has_more` may be false (`service.py:1155` through `1164`). The next request can be empty although the model says `None` means exhausted. The raw key cursor is described as opaque but is not bound to mount, path, or filter flags. | High on the extra page; medium on binding | **Merge with a follow-up ticket** after adding direct page tests. Return `None` when exhaustion is known and document same-query reuse. |
| **[Codex L4-F7] Unbounded inputs multiply resource use.** | Archive requests have no mount-count or total-size guard. Plain `GET /files?order=...&limit=...` has no limit maximum and scans the full tree. Raw `limit=0` does not use `_COUNT_CAP` because that cap is only passed to the Git-aware descent. `.gitignore` bodies are read fully before parsing (`service.py:638`). | High | **Needs a change before merge** for archive cardinality, object-body streaming, and the advertised ignore cap. Add quotas/timeouts for export work and a `.gitignore` size policy. |
| **[Codex L4-F8] Project isolation, permission choice, and mtime units are correct.** | Every archive mount resolves with the request project ID (`service.py:583` and `1200`). Both routes are read-only and use `VIEW_MOUNTS`. New store methods produce epoch milliseconds (`storage.py:331`, `361`, `390`), and ZIP conversion divides by 1000 (`utils.py:97`). No sibling unit mismatch was found. | High | **Merge as-is for these aspects.** Keep an mtime regression test because failures occur after response headers. |

### [Codex] Verification record

- **[Codex]** Fresh remote state was fetched. The reviewed head was
  `3ac6af25160a4b2546c682137f9d3ed5865fabe0`.
- **[Codex]** `git diff --check origin/main...origin/fe-refactor/drive-surfaces` reported no
  backend whitespace errors.
- **[Codex]** Live checks showed Python format, Python lint, API unit, Railway API unit,
  Railway API integration, and Railway API acceptance tests passing.
- **[Codex]** The overall PR was not fully green because web unit/acceptance checks were failing.
  This backend review did not diagnose those frontend failures.
- **[Codex]** The lock diff adds only `pathspec 0.12.1` and `stream-zip 0.0.84`. It does not update
  unrelated locked package versions.
- **[Codex]** No code changed and no GitHub review comments were posted. The only workspace edit is
  this tagged report appended to the supplied handoff.

<!-- CODEX-REVIEW-END -->
