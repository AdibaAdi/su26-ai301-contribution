# Contribution 3: Collect rotated Ray logs in the KubeRay History Server collector

**Contribution Number:** 3
**Student:** Adiba Akter
**Project:** KubeRay (`ray-project/kuberay`)
**Repository:** https://github.com/ray-project/kuberay
**Issue:** [#4830: \[Feature\] Collect rotated logs](https://github.com/ray-project/kuberay/issues/4830)
**Fork:** https://github.com/AdibaAdi/kuberay
**Contribution repo (public):** https://github.com/AdibaAdi/su26-ai301-contribution
**Working branch:** [`feat/4830-collect-rotated-logs`](https://github.com/AdibaAdi/kuberay/tree/feat/4830-collect-rotated-logs)
**Status:** Phase I complete. Phase II investigation, reproduction and solution planning complete and recorded below. Work in progress on the branch is not documented here yet, and no upstream pull request exists.

---

## Contribution Summary

This contribution targets KubeRay issue #4830: the History Server log collector does not
capture Ray's *rotated* log files, so log content that rotates away while a cluster is
still running is lost before it ever reaches object storage.

For Phase I, I forked the repository, created a local branch from the latest upstream
`master`, introduced myself on the issue, and completed a read-only investigation of the
collector code. The issue author asked me to share a design before writing code because
the task is not straightforward, so I posted a proposed design publicly on the issue with
the author and project members tagged. Feedback is pending, and Phase II reproduction work
continues in the meantime.

The sections up to and including the Phase I completion checklist are Phase I work and are
left as they were written, including the findings I labelled preliminary at the time. The
Phase II section that follows the checklist records what I went on to confirm, what I had
to correct, how I reproduced the loss, what the root cause turned out to be, and the
implementation and validation plan that came out of it. No upstream pull request exists,
and I am not claiming the issue is solved.

---

## Why I Chose This Issue

I picked #4830 because it is the kind of problem I keep running into from the other
side. My background is Python, backend services, and testing, and most of my recent
interest has been in AI infrastructure, the unglamorous layer that decides whether an ML
job is debuggable at 3 a.m. Log collection for a distributed Ray cluster is exactly that
layer. It is not a flashy feature, but if it silently drops data, every downstream
investigation gets harder.

It is also a deliberate stretch. My first two contributions were Java (OpenSearch k-NN)
and Python/Rust (Lance). This one is Go and Kubernetes, which I have read but not shipped
in. The nice thing about this particular issue is that the hard part is *not* the
language: the hard part is reasoning about file lifecycle, idempotency, and restart
safety, which is ordinary backend and testing thinking that I already have. That gives me
a realistic path to learn Go idioms (goroutines, tickers, `filepath.WalkDir`, table-driven
tests) and the KubeRay sidecar/volume model without the correctness reasoning also being
new to me at the same time.

Finally, the scope discipline appealed to me. The reporter explicitly said the task is not
straightforward and asked for a design first. That is a good forcing function against my
usual instinct to start editing code, and it matches the lesson I wrote down at the end of
Contribution 1: open a scoping conversation with maintainers *before* planning an
implementation, not after.

---

## Problem Summary

Ray rotates its log files by default (512 MB `maxBytes`, five backups), renaming
`raylet.out` to `raylet.out.1`, then `.1` to `.2`, and deleting the oldest backup once
the backup count is exceeded. The KubeRay History Server log collector uploads the
contents of the session log directory mainly at process shutdown, plus previous-session
logs staged under `prev-logs`, so for a long-running cluster a rotated segment can be
created and deleted between two upload points and never reach object storage. The
requested change is a path that uploads *completed* rotated segments continuously while
the cluster is still alive. The design difficulty is that the same physical segment
changes filename as it ages, so the collector has to recognise it across renames without
uploading duplicates and without re-uploading everything after a restart.

---

## Why This Matters for Long-Running Distributed AI Workloads

The reporter's stated motivation is that rotation exists because the local filesystem can
run out of disk, but the History Server writes to object storage, which has no such limit,
so there is no good reason to lose the rotated content. In practice this bites hardest on
exactly the workloads KubeRay exists to serve: multi-day training runs, long-lived Ray
Serve deployments, and batch inference clusters that stay up for weeks. Those are the
clusters that generate enough log volume to rotate in the first place, and they are also
the ones where a postmortem needs the *early* logs, the ones that rotated away first, to
explain a failure that surfaced much later. A collector that only reliably captures the
final window of logs is most reliable precisely when it is least needed.

---

## Issue Selection Rationale

| Consideration | Assessment |
|---|---|
| Problem understanding | The issue clearly describes a gap in the current History Server collector: rotated log segments may be deleted before they are uploaded to object storage. My problem summary and codebase investigation trace that behavior to the collector's shutdown and previous-session recovery paths. |
| Scope | The issue is non-trivial, but the expected change appears bounded. The proposed work adds a new collection path alongside the existing shutdown and `prev-logs` behavior rather than redesigning the entire collector. The main technical risk is identifying rotated segments reliably across filename changes. |
| Skill alignment | The work builds on my experience with backend systems, file lifecycle behavior, idempotency, restart safety and test design. It also gives me a practical opportunity to strengthen my Go and Kubernetes experience. |
| Repository activity and issue availability | KubeRay and the `historyserver` area are actively maintained. Issue #4830 is open and unassigned, with no pull request currently linked as its implementation. PR #4983 touches related collector code, so I will track it closely to avoid conflicting work. |
| Available technical context | The issue links directly to Ray's log-rotation documentation, and the repository contains existing shutdown, recovery and persistence logic that provides a concrete starting point. Related issue #4825 and PR #4983 also provide useful architectural context. |
| Development and contribution workflow | KubeRay provides clear contribution guidelines and a dedicated `historyserver/DEVELOPMENT.md` guide covering the local Kind, MinIO and History Server setup. This gives me a documented path for reproducing the behavior and validating a solution. |

---

## Repository Activity and Claimability Evidence

Verified on 2026-07-26 against my local clone of `ray-project/kuberay`, whose `upstream`
remote was last fetched on 2026-07-22 at `6fe0223c`. Because the clone is a few days
stale, the commit windows below are relative to that fetch, not to today.

| Signal | Value |
|---|---|
| `upstream/master` HEAD | `6fe0223c` ("Rename TargetClusterChanged for clarity (#5022)") |
| Most recent commit date | 2026-07-22 |
| Commits on `master` in the prior 30 days | 31 |
| Distinct authors in the prior 90 days | 35 |
| Commits touching `historyserver/` in the prior 90 days | 19 |
| Issue state | Open, filed 2026-05-12 by `dentiny` |
| Labels | `P2`, `enhancement` (a `triage` label was applied at filing) |
| Assignee | None |
| Linked branches / PRs on the issue | None |

Commands used, for anyone reproducing this:

```bash
git fetch upstream
git log --oneline --since=2026-06-22 upstream/master | wc -l
git log --since=2026-04-22 --format='%aN' upstream/master | sort -u | wc -l
git log --oneline --since=2026-04-22 upstream/master -- historyserver/ | wc -l
```

The `historyserver/` subtree specifically is under active development, which is a
double-edged signal: the area is alive and maintained, but my change will have to be
rebased against moving code (see the Related Issue/PR Context section).

### Fork and branch state

```bash
git remote -v
#   origin    https://github.com/AdibaAdi/kuberay.git
#   upstream  https://github.com/ray-project/kuberay.git

git rev-list --count upstream/master..feat/4830-collect-rotated-logs
#   0    -> branch created from upstream master, zero commits ahead
```

The branch exists locally and is currently identical to upstream `master`. It has not been
pushed, because there is nothing on it yet.

---

## Community and Maintainer Communication

All four links below are public comments on issue #4830 and can be opened directly.

1. **Earlier project-member signal that the direction is wanted:**
   https://github.com/ray-project/kuberay/issues/4830#issuecomment-4523841501
   A project member (`JiangJiaWei1103`) engaged with the request before I was involved.
   This predates my involvement and is what convinced me the feature was worth picking up
   rather than a request that had already been declined.

2. **My introduction on the issue:**
   https://github.com/ray-project/kuberay/issues/4830#issuecomment-5052716814
   I said I wanted to work on the issue and described how I planned to start.

3. **The issue author asking me to share a design first:**
   https://github.com/ray-project/kuberay/issues/4830#issuecomment-5052727165
   `dentiny`, who filed the issue, made the point that this is not a straightforward task
   and that a design should be discussed before implementation. I took that seriously
   rather than treating it as optional.

4. **My detailed design comment:**
   https://github.com/ray-project/kuberay/issues/4830#issuecomment-5052938252
   This is the design summarised in the "Proposed Direction" section below. I posted it
   publicly on the issue with the author and project members tagged, so the discussion
   happens in the open where anyone reviewing can see it.

**Current state of the conversation:** feedback on the full design is pending. To be
explicit about what that does and does not mean:

- The issue is **not** assigned to me, and I have not asked for it to be.
- Nobody has approved my proposed design. Pending feedback is exactly that: pending. I am
  not treating the absence of a reply as agreement.
- The only thing endorsed so far is the general idea of collecting rotated logs, in the
  earlier project-member comment (link 1) and in the issue author's own framing.
- Phase II reproduction and validation continue in the meantime, because reproducing the
  loss and measuring the rotation behaviour is useful regardless of which mechanism is
  eventually chosen. Any feedback that arrives will be incorporated into the design and
  recorded here.

---

## Preliminary Codebase Orientation

This section is a **read-only** investigation of `historyserver/` at `6fe0223c`. I have
not run the collector, so these are code-reading findings, not observed runtime behaviour.

### How the collector is structured today

`RayLogHandler.Run` (`historyserver/pkg/collector/logcollector/runtime/logcollector/collector.go:49`)
starts a small set of goroutines and then blocks on a stop channel:

- `WatchPrevLogsLoops()` runs on **both** head and worker nodes. It scans
  `/tmp/ray/prev-logs` on startup and then uses three `fsnotify` watchers to pick up new
  session/node directories, so leftover logs from a previous run get uploaded.
- Head-only goroutines: `WatchSessionLatestLoops()`, `FetchAndStoreClusterMetadata()`,
  `FetchAndStoreTimezone()`, and `PollAdditionalEndpointsPeriodically()`.
- On stop, `Run` calls `processSessionLatestLogs()` (both roles), then
  `processAdditionalEndpoints()` (head only), then closes `ShutdownChan`.

### Preliminary finding 1: the live-session upload really is shutdown-shaped

`processSessionLatestLogs` (`collector.go:91`) resolves the `session_latest` symlink,
reads the node ID from `/tmp/ray/raylet_node_id`, and `filepath.WalkDir`s the session
`logs/` directory, uploading every regular file it finds via `processSessionLatestLogFile`
(`collector.go:176`), which does a whole-file `os.ReadFile` followed by
`Writer.WriteFile`. As far as I can tell from reading `Run`, this function is called from
exactly one place: the shutdown path. There is no ticker, no watcher, and no other caller
that uploads live-session log content while the cluster is running. This matches the
reporter's description of the mechanism.

### Preliminary finding 2: there is already a dedupe pattern worth reusing

The `prev-logs` path solves the duplicate-upload problem in a way that looks directly
reusable. `processPrevLogFile` (`collector.go:615`) uploads a file and then `os.Rename`s
it into `/tmp/ray/persist-complete-logs/{sessionID}/{nodeID}/logs/...`, and
`isFileAlreadyPersisted` (`collector.go:517`) checks for the file's presence in that
directory before uploading. The marker lives on the shared volume, so it survives a
collector restart. Two details matter for #4830: the marker path is derived from the
*filename*, which is exactly the thing that changes when a backup is renamed from `.1` to
`.2`; and `processPrevLogsDir` deletes the whole node directory when it finishes
(`collector.go:607`), which is fine for a dead session but would be wrong for a live one.

### Preliminary finding 3: there is an existing periodic-loop pattern to copy

`PollAdditionalEndpointsPeriodically` (`poll.go:26`) is the shape a rotated-log scanner
should follow: do one pass immediately, then `time.NewTicker(interval)` in a
`for { select { case <-r.ShutdownChan: ...; case <-ticker.C: ... } }` loop. If I add a
periodic scanner, following this structure keeps it consistent with code that already
exists rather than inventing a new lifecycle.

### Preliminary finding 4: the periodic-push plumbing exists but is unused

`RayLogHandler` declares `PushInterval`, `LogBatching`, `LogFiles`, `logFilePaths`, and
`LogDir` (`collector.go:23-47`). `PushInterval` is fed by the `--push-interval` flag in
`historyserver/cmd/collector/main.go` (default one minute) and plumbed through
`runtime.NewCollector` (`runtime.go:18`), but grepping the non-test sources shows it is
only ever *assigned*, never read by the log-collector loops. `r.LogDir` appears in `Run`
only in a commented-out line (`collector.go:50`). This is preliminary and I want to
confirm it upstream, but if it holds, there is already a configured interval
looking for a consumer, and I should ask whether reusing `--push-interval` is preferred
over adding a new flag.

### Preliminary finding 5: what Ray actually does on rotation

From the Ray logging documentation the issue author wrote and linked in the issue:
rotation is on by default at 512 MB `maxBytes` with a `backupCount` of five, backups are
indexed (`raylet.out.1`), and both are configurable via `RAY_ROTATION_MAX_BYTES` and
`RAY_ROTATION_BACKUP_COUNT`. Ray deletes the oldest backup once the backup count is
exceeded, which is the deletion race at the centre of this issue. Ray's documentation
indicates that rotation behavior may differ by log component. The exact filenames and
components that rotate in the KubeRay test image will be verified empirically during
Phase II, along with the exact rename ordering, which I have not yet observed.

### Proposed direction (as posted on the issue, not yet approved)

A periodic scanner that:

- handles **completed** rotated files only (`*.N` suffixes), and never touches the active
  file, so a partially written file is never uploaded as though it were final;
- records successful uploads on the shared volume, using the existing
  `persist-complete-logs` marker idea, so restarts do not re-upload;
- identifies a segment by something stable across renames rather than by filename alone,
  so a rename from `.1` to `.2` does not produce a second copy of the same bytes;
- leaves failed uploads unmarked so they are retried on the next pass;
- leaves the existing shutdown and `prev-logs` paths completely unchanged.

The known limitation, which I stated in the design comment rather than hiding, is that a
scanner cannot guarantee completeness: if rotation and deletion both happen faster than
the scan interval, segments will still be missed. That is a tuning trade-off, not a bug I
can design away, and reviewers may have a stronger opinion about it.

---

## Likely Files and Modules

Identified by reading the tree at `6fe0223c`. Line counts are for orientation only, and
the "likely role" column is my expectation, not a completed change.

| Path | Lines | Likely role in the change |
|---|---:|---|
| `historyserver/pkg/collector/logcollector/runtime/logcollector/collector.go` | 779 | Primary. Holds `RayLogHandler`, `Run`, `processSessionLatestLogs`, `processPrevLogFile`, and `isFileAlreadyPersisted`. A new scanner goroutine would be started from `Run` and would reuse or generalise the persisted-marker helpers. |
| `historyserver/pkg/collector/logcollector/runtime/logcollector/poll.go` | 142 | Reference pattern for a ticker loop driven by `ShutdownChan`. Possibly the file a new scanner lives beside, to keep `collector.go` from growing further. |
| `historyserver/pkg/collector/logcollector/runtime/logcollector/collector_test.go` | 241 | Existing test harness: `MockStorageWriter` (an in-memory `storage.StorageWriter`) and `setupRayTestEnvironment`, which builds `prev-logs` / `persist-complete-logs` trees under a temp root. New rotation tests should extend this rather than build a parallel harness. |
| `historyserver/pkg/collector/logcollector/runtime/runtime.go` | 60 | `NewCollector` wires config into `RayLogHandler`; any new interval or toggle is threaded through here. |
| `historyserver/pkg/collector/types/types.go` | n/a | `RayCollectorConfig`, where a new configuration field would be declared. |
| `historyserver/cmd/collector/main.go` | 192 | Collector entrypoint and flag parsing; already defines `--push-interval` and reads `RAY_COLLECTOR_*` environment variables, so it is the natural place for scan configuration. |
| `historyserver/pkg/utils/constant.go` | 38 | `GetTmpRayRoot`, `GetRayPrevLogsPath`, `GetRayPersistCompletePath`, `GetRaySessionLatestPath`, `GetRayNodeIDPath`. Every path the collector touches resolves through here, including the `RAY_TMP_ROOT` override the tests rely on. |
| `historyserver/pkg/utils/utils.go` | 174 | `GetLogDirByNameID` builds the object key layout `{root}/{cluster}_{ns}/{session}/logs/{nodeID}/...`; rotated segments must land in the same layout so the History Server UI can read them. |
| `historyserver/pkg/storage/interface.go` | 26 | `StorageWriter` is only `CreateDirectory` + `WriteFile`. Worth confirming upstream whether whole-file rewrites are acceptable for large rotated segments or whether an append/streaming concern needs raising. |
| `historyserver/DEVELOPMENT.md` | n/a | The kind + MinIO + History Server walkthrough I will follow in Phase II to actually observe rotation. May need a note if new configuration is added. |
| `historyserver/config/`, `historyserver/test/e2e/` | n/a | Collector sidecar configuration and end-to-end tests; likely touched only if a new flag or environment variable is introduced. |

---

## Related Issue / PR Context

- **[PR #4983](https://github.com/ray-project/kuberay/pull/4983)** modifies node ID
  handling and leftover-log handling in the same collector area. This overlaps with the
  code I would be changing: node ID resolution feeds the storage key built by
  `GetLogDirByNameID`, and "leftover logs" is the `prev-logs` path whose dedupe mechanism
  I want to reuse. Whatever I implement has to be rebased on top of it, and if it changes
  how the persisted-marker directory works, my design changes with it. I am treating this
  PR as a dependency to track, not a blocker.
- **[Issue #4825](https://github.com/ray-project/kuberay/issues/4825)**, "Question:
  what's the usage pattern for history server", is referenced from #4830 by the same
  reporter. It is useful background on how the History Server is expected to be used,
  which matters for deciding whether a lossy-but-cheap scanner is acceptable.
- **[Ray log rotation documentation](https://docs.ray.io/en/latest/ray-observability/user-guides/configure-logging.html#log-rotation)**
  is the authoritative description of the rotation behaviour. It was written by the issue
  reporter and linked from the issue body.

---

## Preliminary Definition of Done / Acceptance Criteria

These are the outcomes I proposed. They are **not** agreed criteria. Every one of them is
still subject to confirmation by the issue author and project members, and criterion 1 in
particular is a best-effort statement rather than a guarantee. Phase II is where I expect
to validate or correct several of them empirically.

1. Completed rotated log segments are uploaded to object storage before Ray deletes them,
   under ordinary operating conditions (that is, when rotation is not faster than the scan
   interval).
2. The currently active log file is never uploaded as though it were a completed segment.
3. Renaming a backup from `.1` to `.2` does not produce duplicate stored content for the
   same physical segment.
4. A segment that was successfully uploaded is not uploaded again after the collector
   restarts.
5. A segment whose upload failed remains eligible for retry on a later pass.
6. Existing graceful-shutdown collection and `prev-logs` recovery continue to behave
   exactly as they do today.
7. Both head-node and worker-node logs are handled, consistent with `WatchPrevLogsLoops`
   already running on both roles.
8. Tests cover: a rotation cycle, duplicate detection across renames, restart behaviour,
   upload failure and retry, and a regression test that the shutdown path still uploads
   what it uploads today.

Criteria 2 through 6 are the ones I would defend hardest, because they are about not
breaking or double-writing anything. Criterion 1 is the one most likely to need rewording
once there is a shared view of how much loss is acceptable.

---

## Current Risks and Open Questions

**Risks**

- *Rotation is faster than the scan.* A periodic scanner is inherently best-effort. If
  512 MB segments rotate within one interval, content is still lost. I flagged this in the
  design comment; reviewers may prefer a different mechanism (inotify on the log
  directory, or hard-linking on rotation) that I have not designed.
- *Identifying a segment across renames.* Filename is not a stable identity. Inode number,
  size, and content hash are all candidates, and each has a cost or a failure mode
  (hashing 512 MB per scan is expensive; inode numbers are reused after deletion). I do
  not have a settled answer yet, and this is the single most important thing to resolve
  in Phase II.
- *Collision with PR #4983.* Concurrent changes to the same collector area mean rebase
  work and possibly a design change.
- *Whole-file reads.* The current upload path does `os.ReadFile` into memory. Doing that
  per rotated segment in a sidecar with a memory limit may be a real problem at Ray's
  default 512 MB rotation size. I have not measured this.
- *Scope growth.* This is a P2 enhancement that the issue author called non-trivial. If
  the design conversation expands it, I need to renegotiate scope on the issue in time to
  still ship something.

**Open questions for the issue author and project members**

1. Is a periodic scanner the right mechanism, or is there a preference for filesystem
   notifications or a Ray-side hook?
2. Should the scan interval reuse the existing `--push-interval` flag, or should it be a
   separate flag/environment variable?
3. Should rotated-log collection be on by default, or opt-in behind a flag?
4. What is the preferred identity for a rotated segment across renames, given the cost
   trade-offs above?
5. Is a whole-file read-and-write acceptable for 512 MB segments, or does the
   `StorageWriter` interface need a streaming path first?
6. Should coordination with PR #4983 happen by waiting for it to merge, or by building on
   its branch?

---

## Phase I Completion Checklist

- [x] Issue identified and read in full, including the linked Ray rotation documentation
- [x] Issue selection rationale documented across problem understanding, scope, skill
      alignment, repository activity, available context and contribution workflow
- [x] Repository activity and claimability verified against the local clone, with commands
      recorded
- [x] Repository forked to `AdibaAdi/kuberay`
- [x] Local branch `feat/4830-collect-rotated-logs` created from the latest upstream
      `master`
- [x] Introduction posted on the issue
- [x] Issue author's request for a design read and honoured
- [x] Read-only investigation of the collector completed
- [x] Detailed design posted publicly on the issue with the author and project members
      tagged
- [x] Related PR (#4983) and related issue (#4825) identified
- [x] Likely files and modules identified with concrete paths
- [x] Preliminary acceptance criteria drafted
- [x] Risks and open questions written down
- [x] This Phase I document written and the root README updated
- [ ] Feedback on the design: pending; will be incorporated if received
- [ ] Issue assignment: not requested, not granted

---

## Phase II: Investigation, Reproduction and Solution Planning

### Development Environment

All Phase II work happens on
[`feat/4830-collect-rotated-logs`](https://github.com/AdibaAdi/kuberay/tree/feat/4830-collect-rotated-logs)
in my fork, `AdibaAdi/kuberay`, which is pushed and tracked as
`origin/feat/4830-collect-rotated-logs`. The branch was created from upstream `master`, and
since Phase I it has been rebased onto a newer upstream commit; upstream has continued to
move, and `ray-project/kuberay` `master` is at `dc3afe89` (2026-07-31) as of this writing,
which is not yet an ancestor of my branch. Rebasing again before any pull request is
therefore part of the plan rather than an afterthought, and the section on what changed
underneath me explains why that matters more than usual here.

The History Server is its own Go module: `historyserver/go.mod` declares
`module github.com/ray-project/kuberay/historyserver` and requires Go 1.26, so every build,
vet and test command in this section is run from the `historyserver/` directory rather than
the repository root. The collector I am changing is the sidecar described by
`historyserver/config/raycluster.yaml`: a `collector:v0.1.0` container that shares an
`emptyDir` volume with the `rayproject/ray:2.52.0` container, both mounting it at the path
in `RAY_TMP_ROOT` (`/tmp/ray`), with `S3_ENDPOINT` pointing at the in-cluster MinIO service
and `--ray-root-dir=log` setting the object-storage prefix. That shared-volume shape is the
single most important environmental fact for this issue: the collector does not receive
Ray's logs over a socket or an API, it reads the same directory tree Ray writes, so
everything about rotation reaches the collector as ordinary filesystem events on a
filesystem it can also write to.

My Phase II reproduction and investigation were done at the Go package level on my macOS
development host, against real temporary directories created by `t.TempDir()`, with real
files, real renames, real hard links and real link counts, rather than inside a Kubernetes
cluster. That was a deliberate choice and not only a convenience one. The behaviour in
question is filesystem lifecycle — rename, unlink, link count, inode reuse — and a
`kind` cluster reproduces none of it more faithfully than the local filesystem does; what a
cluster adds is the sidecar wiring, the object-store round trip and the History Server's
retrieval path. I want that end-to-end confirmation too, and the walkthrough in
`historyserver/DEVELOPMENT.md` (kind, the KubeRay operator via Helm,
`make -C historyserver localimage-build`, `kind load docker-image` for `collector:v0.1.0`
and `historyserver:v0.1.0`, then `historyserver/config/minio.yaml`) is the route I intend to
follow for it. I have not run that end-to-end path yet, and I say so plainly rather than
implying a cluster-level result I do not have.

Two concrete environment problems came up and were worth the time they cost.

The first was developing inode-level code on macOS. Rotated-log capture depends on
`syscall.Stat_t`, and the device, inode and link-count fields of that struct differ in both
width and signedness across Unix targets, so code written naively against the Linux field
types does not compile on darwin — and the collector image is Linux, so I could not simply
target one and hope. I resolved it by splitting the platform-specific part into a single
tiny function behind a build constraint: `rotated_stat_unix.go` (`//go:build unix`) provides
`inodeFromFileInfo`, which extracts the device/inode pair and link count through a generic
`statNumber` helper that widens all of those integer types to `uint64`, and
`rotated_stat_other.go` (`//go:build !unix`) provides a stub that returns an explicit
"rotated log capture is unsupported on this platform" error instead of failing to build.
The practical benefit is that the capture logic itself is platform-neutral and I can develop
and test it on the same host I write it on: `go vet` is clean, and the capture tests pass on
darwin while the production target stays Linux.

The second was that I could not settle Ray's rotation naming by observation. Ray's defaults
are a 512 MB `maxBytes` with five backups, which is not something to wait for on a laptop,
and the KubeRay manifests pin `rayproject/ray:2.52.0` whose components are a mix of Python
and C++ writers. Turning the rotation thresholds down with `RAY_ROTATION_MAX_BYTES` and
`RAY_ROTATION_BACKUP_COUNT` makes rotation happen sooner but still only tells me what one
image did on one run, which is a weak basis for a regular expression that decides what the
collector is allowed to touch. I resolved it by tracing Ray's implementation instead of
guessing from a sample: Python's `RotatingFileHandler` appends `.1`, `.2`, and Ray patches
spdlog so its C++ components agree, in
`thirdparty/patches/spdlog-rotation-file-format.patch`, which rewrites `calc_filename` so
that `("logs/mylog.txt", 3)` produces `logs/mylog.txt.3` rather than spdlog's default
`logs/mylog.3.txt`. I verified that patch upstream in `ray-project/ray` rather than taking
the documentation's word for it. Ray therefore has one rotation naming convention across
languages — active name, then a dot, then a positive index — which is what
`rotationBackupRe` encodes, and the cascade can then be replayed deterministically in tests
with `os.Rename` and `os.Remove` instead of waiting on a real writer.

### Reproducing the Problem

These steps reproduce the gap at the package level on a Unix host, without a cluster. They
describe the unmodified collector, so they can be run against upstream `master` as well as
against my branch.

1. Clone the fork and check out the working branch, or check out upstream `master` if you
   want the behaviour without any of my changes:
   `git clone https://github.com/AdibaAdi/kuberay.git && cd kuberay && git checkout feat/4830-collect-rotated-logs`.
2. Change into the History Server module, which is where all Go commands must run:
   `cd historyserver`. Confirm the toolchain matches `go.mod` (Go 1.26).
3. Establish how live-session logs reach storage today, by finding every caller of the
   upload routine rather than assuming: `git grep -n "processSessionLatestLogs" -- .`, or,
   with an `upstream` remote added and fetched,
   `git grep -n "processSessionLatestLogs" upstream/master -- historyserver/` to check the
   unmodified upstream tree. There is exactly one call site,
   `historyserver/pkg/collector/logcollector/runtime/logcollector/collector.go:94` on
   upstream `master`, and it sits in `RayLogHandler.Run` immediately after the `<-stop`
   receive.
4. Confirm that no timer drives an alternative path, with the same grep for
   `PushInterval`. Every hit is an assignment — `cmd/collector/main.go`,
   `runtime/runtime.go`, `types/types.go`, the struct field in `collector.go`, and the
   storage backends — and none of the log-collector loops ever reads it.
5. Build a Ray-shaped tree in a temporary directory: a session directory containing
   `logs/`, a `session_latest` symlink pointing at it, and a node ID file, so that the
   collector's `RAY_TMP_ROOT` override resolves the same way it does in the sidecar. The
   existing `setupRayTestEnvironment` helper in `collector_test.go` builds the same shape
   for the `prev-logs` and `persist-complete-logs` trees.
6. Write a log file, for example `logs/raylet.out`, with recognisable content.
7. Replay one rotation cycle exactly as Ray performs it: rename `logs/raylet.out` to
   `logs/raylet.out.1`, let the writer recreate `logs/raylet.out`, and repeat until the
   backup count is exceeded, at which point the oldest backup — the file holding the
   earliest content — is unlinked.
8. Drive the collector's upload path with an in-memory writer. `MockStorageWriter` in
   `collector_test.go` records every object key and body handed to
   `storage.StorageWriter.WriteFile`, so the set of keys it holds is a direct answer to the
   question "what would have reached object storage?".
9. Compare the keys written against the segments that existed during the run. The content
   that was unlinked in step 7 has no key, because by the time anything uploads, the only
   files that exist are the ones the walk can still see.

The same conclusion can be reached from the other direction, which is why I did both:
`processSessionLatestLogs` resolves `session_latest` with `filepath.EvalSymlinks` and then
`filepath.WalkDir`s the resulting `logs/` directory, so what it uploads is by definition
whatever is present at the moment it runs. A segment that Ray unlinked an hour earlier is
not in that tree and cannot be.

### Expected and Observed Behavior

What I expect, and what the issue asks for, is that a rotated log segment that Ray has
finished writing is durable. Once `raylet.out` becomes `raylet.out.1`, that file will never
be appended to again; it is complete. It should reach object storage under the node's log
prefix within a bounded delay, exactly once, and be retrievable afterwards through the
History Server's node-log listing alongside the segments that happened to survive until
shutdown. Whether the cluster later runs for another two weeks should not change whether
that content still exists.

What actually happens is that durability is tied to survival. The collector's live-session
upload runs once, at shutdown, and `prev-logs` recovery runs for sessions that have already
ended. A rotated segment is durable only if it is still on disk when one of those two moments
arrives. Under Ray's defaults, a cluster that rotates through its five backups discards the
oldest one every time a new rotation happens, so on a long-running cluster the earliest
content is systematically the content most likely to be gone. The failure is silent: no
error is logged, no upload fails, and the objects that do land in storage look complete.
That silence is what makes this worth fixing — an operator has no signal that the first
hours of a multi-day run were never captured until they go looking for them.

It is worth being precise about which loss I am describing. I am not describing the tail of
the active file, which is always a partial window; I am describing segments that were
complete, closed and never touched again, which is the case where nothing about the file's
state prevented capture.

### Investigation and Root Cause

The collector lives in the Go module under `historyserver/`, and the code that matters for
this issue is in one package,
`historyserver/pkg/collector/logcollector/runtime/logcollector`, with supporting path and
storage helpers in `historyserver/pkg/utils` and `historyserver/pkg/storage`.
`RayLogHandler.Run` in `collector.go` is the lifecycle: it resolves `prev-logs` and
`persist-complete-logs` through `utils.GetRayPrevLogsPath` and
`utils.GetRayPersistCompletePath`, starts `WatchPrevLogsLoops` and `PollActiveSessionChanges`
on every node plus `WatchSessionLatestLoops`, `FetchAndStoreClusterMetadata`,
`FetchAndStoreTimezone` and `PollAdditionalEndpointsPeriodically` on the head, then blocks
on the stop channel. After the stop signal it calls `processSessionLatestLogs`, then
`processAdditionalEndpoints` on the head, then closes `ShutdownChan`. The upload itself is
`processSessionLatestLogFile`: it computes a path relative to the session `logs/` directory,
builds the object key with `clusterlogs.LogsDir`, reads the whole file with `os.ReadFile`
and hands a `bytes.NewReader` to `storage.StorageWriter.WriteFile`.

Phase I's preliminary findings 1 and 4 both hold, and I confirmed them against upstream
`master` at `dc3afe89` rather than against the older commit I read during Phase I. There is
one caller of `processSessionLatestLogs`, in the shutdown path, and `PushInterval` is
plumbed from the `--push-interval` flag all the way into `RayLogHandler` without any
log-collector loop ever reading it. Two other Phase I statements did not survive contact
with current upstream, and I would rather correct them here than leave them standing.
`GetLogDirByNameID` no longer exists: the object-key layout is now built by
`historyserver/pkg/storage/clusterlogs`, whose `LogsDir` produces
`<root>/cluster-history/<ownerKind>/<namespace>[/<ownerName>]/<cluster>/<session>/<node>/logs`
and which distinguishes `RayCluster` from `RayJob` and `RayService` owners, so any new
upload path must take owner kind and owner name into account or it will write to the wrong
prefix. And PR #4983, which Phase I listed as a dependency to track, has merged as
`942ed40a`; running `git log --oneline 6fe0223c..upstream/master -- historyserver/` also
shows `4efe66e3` (#4918), which is what changed the log file structure, `10bfea78` (#5050)
and `dc3afe89` (#5067), the last of which touches graceful collector exit — directly
adjacent to the shutdown sequencing this work has to fit into. The `historyserver/` subtree
moving this fast is the practical risk to my change, more than any single one of those
commits is.

One Phase I open question resolved itself in my favour on inspection. I had asked whether
`StorageWriter` needs a streaming path before large segments can be uploaded;
`historyserver/pkg/storage/interface.go` declares
`WriteFile(file string, reader io.ReadSeeker) error`, and an `*os.File` satisfies
`io.ReadSeeker`. The interface already supports streaming — the existing shutdown path
simply chooses `os.ReadFile` and wraps the bytes. A new upload path can open the file and
pass the handle, so a 512 MB segment need not be held in the sidecar's memory, and no
interface change has to be negotiated first.

The visible symptom is that rotated segments go missing. The root cause is two separate
things that compound, and only the first is obvious.

The collector's collection is event-driven, but it is driven by the wrong events. Its
triggers are process shutdown and the appearance of directories under `prev-logs`. Rotation
is neither of those. Nothing in the collector is watching the live session's `logs/`
directory for the one event that actually marks a segment as complete — the rename of the
active file to `.1` — so the window between that rename and the segment's eventual unlink is
unobserved. Adding a periodic scan closes part of that window, which is what I proposed in
Phase I.

The deeper cause is that the collector's unit of identity is a filename, and rotation makes
a filename a slot rather than a name. `raylet.out.1` refers to a different physical file
after every cascade. The existing `prev-logs` deduplication is built on exactly that
assumption — `isFileAlreadyPersisted` checks for a marker whose path is derived from the
file's name — which is sound for a dead session whose files never move again, and unsound
for a live one. So a filename-keyed periodic scanner does not merely miss segments when it
scans too slowly; it actively confuses distinct segments, and can both skip content it has
never uploaded and re-upload content it already has. Underneath even that is the reason the
loss is irreversible in the first place: nothing holds a reference to the bytes. A directory
entry is the only thing keeping the content reachable, and when Ray unlinks the last one the
kernel is free to reclaim it. A scanner that arrives late has nothing left to read.

Naming that third layer is what changed my design, because it also points at the remedy. The
collector shares the filesystem with Ray, so it can hold a reference of its own: a hard link
into a staging directory is a second directory entry for the same inode. Creating one costs
no copy and no additional data blocks, and from the moment it exists the segment survives
both the rename and the unlink. I verified that this is really how the filesystem behaves
rather than assuming it, with focused tests that use no mocks for the filesystem itself:

```bash
cd historyserver
go test ./pkg/collector/logcollector/runtime/logcollector/ \
  -run 'TestStatInode|TestCaptureLinkPinsInode|TestCaptureSurvivesRotationAndDeletion|TestCaptureOfSuccessiveSegmentsAtSamePath' \
  -count=1 -v
```

All four pass on my host. `TestCaptureSurvivesRotationAndDeletion` writes a segment, links
it into staging, renames the original from `.1` to `.2`, and confirms the staged link still
reports the same device/inode pair with a link count of two; it then deletes the rotated
file and confirms the inode is unchanged, the link count has fallen to one, and the content
is still readable in full. `TestCaptureOfSuccessiveSegmentsAtSamePath` covers the identity
question from the other side: it captures one segment at `raylet.out.1`, deletes it, writes
a new one at the same path, and asserts that the two report different inode keys — precisely
because the first link is still holding the old inode alive — and that they receive distinct
capture identifiers and distinct object keys. That second test is the empirical answer to
the Phase I risk I called the single most important thing to resolve.

The consequence of holding a link is one I want stated rather than buried: rotation exists
to bound disk usage, and a pinned link prevents the space from being reclaimed until the
collector releases it. Capture is therefore only safe if it is paired with prompt release
after upload and with a limit on how much may be staged at once. That constraint shaped the
design more than any other.

### Proposed Solution

The plan keeps the existing shutdown walk and `prev-logs` recovery untouched and adds a
rotated-log subsystem beside them, in the same package. Work along these lines is already
under way on the working branch; Phase III is where it gets documented in detail.

The subsystem is organised as a small set of new files in
`historyserver/pkg/collector/logcollector/runtime/logcollector`, each with a single
responsibility. `rotated_names.go` owns naming and identity: `parseBackupName` splits a
rotation backup into its active base name and index using the convention I traced in Ray,
`evaluateCandidate` decides whether a directory entry may be captured at all,
`captureIDGenerator.next` mints a capture ID from wall-clock nanoseconds plus 64 bits of
`crypto/rand` entropy, `captureFileName` and `parseCaptureFileName` attach and recover that
ID, `clusterIdentity.objectKey` builds the storage key on top of `clusterlogs.LogsDir`, and
`stagedEntry` with `newStagedEntry`, `parseStagedPath` and `relDirFor` validate every
component that ends up in a filesystem or object path. `rotated_state.go` holds the
in-memory bookkeeping: `inodeKey`, the `capture` record and its `releasable` predicate, and
a `captureIndex` with `add`, `restore`, `remove`, `observeBase` and `baseObserved`, plus
`validTransition` to allow only the one durable state change a capture may make.
`rotated_fs.go` performs the filesystem operations that make capture real — `statInode`,
`captureLink`, `promoteCapture`, `releaseCapture`, `baseKnownWith`, and the error
classifiers `isUnsupportedLinkError`, `isVanished` and `isWatchResourceExhausted` — while
`rotated_stat_unix.go` and `rotated_stat_other.go` isolate `inodeFromFileInfo` behind the
build constraint described earlier.

Discovery and the event loop live in `rotated_collector.go`, whose `rotatedCollector` runs
`fsnotify` watches over the session's `logs/` tree with a periodic reconcile as a backstop
(`defaultReconcileInterval`, 30 seconds), because a watch cannot see what was already in a
directory when it was added and `fsnotify` can drop events under load. Its `Run`,
`handleEvent`, `scanTree`, `watchAndScan`, `inspectFile` and `capture` methods are the
discovery path, and `reconstructStaging` is what a restarting collector uses to rebuild its
index from the staging volume. Uploading lives in `rotated_uploader.go`, deliberately off
the event-loop goroutine: `uploadWorker` opens the staged link, re-checks that it still
holds the captured inode, and streams the `*os.File` straight into
`storage.StorageWriter.WriteFile`, while `uploadScheduler`, `stagedBytes` and `intakeGate`
handle retry backoff, staging accounting and the pause that stops new captures when the
staging volume is near full. `rotated_runtime.go` supplies the `rotatedSupervisor` that
binds a collector to a concrete session and node, with `ensure`, `preflight` and `shutdown`
covering start-up, environment checks and orderly stop.

Two existing files change. In `collector.go`, `RayLogHandler` gains the supervisor and
`Run` starts rotated collection before the other goroutines — a segment rotated away in the
first seconds of a session is already gone by the time the shutdown walk runs — and stops it
before `processSessionLatestLogs` on the way down, so that the pre-existing data path still
gets its share of the pod's termination grace period. In `historyserver/pkg/utils/constant.go`,
a `GetRayRotatedStagingPath` helper resolves the staging root to `rotated-staging` under
`RAY_TMP_ROOT`, deliberately beside `prev-logs` rather than inside it, because the
`prev-logs` walker uploads and then deletes whole node directories and must never delete
captures that are still draining.

The design rests on four decisions. A completed segment is pinned with a hard link before
anything else is attempted, so capture is decided by whether the link succeeded, not by
whether an upload finished in time. Identity is the capture ID, not the filename and not the
inode — the filename is a reused slot, and an inode number may be handed to an unrelated
file once the last link is dropped, so the inode is only meaningful while the collector's own
link keeps it alive. Progress is durable on disk, encoded as a `pending` or `uploaded`
directory level in the staging path and moved between them with an atomic rename, so a
restart can tell an unsent capture from a finished one without any in-memory state.
And a staged link is released only when both conditions hold: the bytes are in storage, and
the link count has fallen to one, proving Ray has finished with the segment.

Configuration is the part I am least settled on, and it is the first thing I want reviewer
input on: whether rotated collection should be on by default or opt-in, whether the scan
interval and staging watermarks reuse `--push-interval` or get their own flags in
`historyserver/cmd/collector/main.go` and `historyserver/pkg/collector/types/types.go`, and
whether `historyserver/config/raycluster.yaml` should demonstrate the settings.

### Validation Strategy

The mechanism is unusual enough that I want it reviewed before it is polished, so the first
step is not a test at all: the design I posted on the issue described a periodic filename-
based scanner, and what I am now proposing pins segments with hard links. That is a
different mechanism with a different failure surface — it holds disk space that rotation
was meant to reclaim, and it assumes the collector may link the Ray container's files — and
it deserves to be raised on the issue in the same public thread rather than appearing fully
formed in a pull request. The specific questions I owe reviewers are whether pinning is
acceptable at all, how much staging space a sidecar may consume before intake should stop,
and whether the shared-volume assumption holds for deployments that do not use the layout in
`historyserver/config/raycluster.yaml`.

Automated validation is table-driven Go tests beside each new file, in the package's existing
style, exercising the real filesystem rather than a mock of it: real files, real hard links,
real link counts, with only the clock, the ticker, the watcher and the link call itself made
injectable so that races can be driven deterministically instead of waited on. The cases that
matter are a full rotation cycle including the moment the cascade briefly leaves the active
name unlinked; two segments occupying the same rotation filename in turn; a collector restart
that rebuilds its index from the staging volume without re-uploading or duplicating anything;
an upload failure followed by a successful retry to the same object key; refusal to release a
staged link while Ray still holds one; and behaviour when linking is refused outright, which
must warn and skip rather than abort collection.

Regression coverage for the existing paths is non-negotiable, because the strongest argument
for this change is that it adds a path rather than altering one. `collector_test.go` already
exercises the shutdown walk and the `prev-logs` dedupe through `MockStorageWriter` and
`setupRayTestEnvironment` in `TestScanAndProcess`, `TestProcessLogs_SkipSymlinks` and
`TestIsFileAlreadyPersisted`, and those must keep passing untouched, together with
`go vet ./...` and the repository's lint configuration.

End-to-end validation is the piece still outstanding, and it is what the `kind` and MinIO
walkthrough in `historyserver/DEVELOPMENT.md` exists for. The shape of it is to bring up the
cluster, operator, MinIO and the data-generating `RayCluster`, lower `RAY_ROTATION_MAX_BYTES`
and `RAY_ROTATION_BACKUP_COUNT` on the Ray containers so segments rotate in seconds, generate
enough log volume to cycle through several rotations while the cluster stays up, and then
confirm two things that no unit test can: that the captured segments are present in the
`ray-historyserver` bucket for a session that never ended, and that they are visible through
the History Server's node-log listing rather than merely present in storage.

### Risks Carried Into Implementation

Rotation itself is the first source of risk. A cascade renames several files in quick
succession and briefly leaves the active name unlinked before the writer recreates it, so a
backup can legitimately appear at a moment when its base file does not exist; a capture rule
that requires the base to be present right now would skip exactly those segments, which is
why the collector has to remember recently seen base names as well as look for them. Watches
are also not a guarantee — a watch added to a directory cannot see what was already in it,
and `fsnotify` drops events under load — so discovery cannot rest on events alone and needs
the periodic reconcile as a backstop. Losing a race to Ray's delete remains possible and has
to be treated as an ordinary, logged outcome rather than an error condition.

Inode identity is the risk I understand best and trust least. An inode number is only a
meaningful identifier while a link to it exists; once the last link is dropped, the kernel is
free to hand the number to a completely unrelated file. Treating the inode as durable
identity would eventually attach one segment's bookkeeping to another file's bytes. The
design contains this by using the inode purely as a "have I already pinned this exact file?"
key that is valid only while the collector's own link keeps it alive, and by giving each
capture a permanent ID that survives restarts and cannot collide after one, since it is
generated from wall-clock time plus random bytes rather than from a counter.

Hard links carry their own environmental risks. Linking fails with `EXDEV` if the staging
directory is not on the same filesystem as the logs, and with `EPERM` or `EACCES` if the
collector's user may not link the Ray container's files, which is plausible for custom images
whose Ray user differs from the collector's UID. The default manifest puts both on the same
`emptyDir`, so the common case is fine, but the uncommon case has to degrade to a warning and
a skip rather than a crash. The subtler risk is the one noted above: a pinned link keeps disk
space that rotation intended to free, so on a node that is already close to full, capture can
make things worse. Staging watermarks and an `ENOSPC`-triggered pause exist for that, and
they need real numbers agreed with maintainers rather than invented defaults.

Retries and shutdown interact awkwardly, and this is where I expect review to be hardest. An
upload that fails must leave its capture recoverable, which means the capture stays pending
and is retried on a later pass, and a retry must rewrite the same object key so that a
partial or duplicated write converges rather than accumulating. Because the object key
contains the capture ID, that idempotency is structural rather than something the retry logic
has to be careful about. Shutdown is harder: the pod's termination grace period is shared
with the pre-existing shutdown walk, which is the path users already depend on, so the
rotated subsystem can only be given a bounded slice of it. Anything still unsent when that
slice expires stays staged on the volume for the next collector to recover — which works
across a collector restart, but not across pod deletion when the volume is an `emptyDir`,
since the volume dies with the pod. That is a real, remaining hole and I would rather name it
than describe the design as complete.

Object naming and History Server retrieval are the last pair, and they are more coupled than
they first appear. Captured segments must be distinguishable from one another, since several
segments share the rotation name `raylet.out.1` over a session's life, which is why the
capture ID is appended to the stored name. They must also stay flat: `_getNodeLogs` in
`historyserver/pkg/historyserver/reader.go` lists a node's log directory non-recursively
unless the caller supplies a glob containing `**`, so anything filed under an extra directory
level would be invisible to the ordinary listing. Two consequences follow that I have not
resolved and intend to raise. First, `categorizeLogFiles` in the same file buckets log files
by substring and suffix, so a captured `worker-abc.out.1.rotated.<id>` no longer ends in
`.out` and would be categorised as `internal` rather than `worker_out`, while
`raylet.out.1.rotated.<id>` still matches on `raylet.` and lands correctly — meaning the
naming scheme has a visible effect on how the dashboard groups files. Second, the existing
shutdown walk uploads whatever rotated backups still exist when it runs, under their plain
rotation names and without filtering, so a segment that survives to shutdown could be stored
twice: once as a capture and once under its rotation name. The keys are disjoint, so nothing
is overwritten and no data is lost, but the duplication is real and a reviewer may reasonably
prefer that one of the two paths yield to the other.

---

## Next Steps

### Completed in Phase I

Fork and branch setup, issue-selection evidence, community introduction, the read-only
code investigation summarised above, and the public design comment. This document is the
Phase I deliverable.

### Phase II (complete, recorded above)

Phase II proceeded without waiting for feedback, because reproducing the loss and
characterising the rotation behaviour is useful evidence whichever mechanism is eventually
chosen. The full account is in the Phase II section above. Against the plan I wrote at the
end of Phase I:

1. The environment work was done at the Go package level rather than in a cluster, for the
   reasons given under Development Environment. The `kind` and MinIO walkthrough in
   `historyserver/DEVELOPMENT.md` remains the route for end-to-end confirmation and has not
   been run yet.
2. Ray's rotation naming was settled by tracing Ray's own implementation, including the
   spdlog rotation patch, rather than by observing one image's behaviour with reduced
   `RAY_ROTATION_MAX_BYTES` and `RAY_ROTATION_BACKUP_COUNT` values.
3. The loss was reproduced at the package level and traced to a single shutdown-time call
   site; the cluster-level demonstration in MinIO is still outstanding.
4. Preliminary findings 1 and 4 were both confirmed against current upstream `master`.
5. Segment identity across renames was resolved with evidence: a hard-link pin plus a
   generated capture ID, with the inode used only while the collector's own link keeps it
   alive.
6. PR #4983 has merged upstream, so the question of waiting for it or building on it no
   longer applies; the relevant upstream churn is now #4918, #5050 and #5067.
7. Which components rotate under which filenames was answered from Ray's source rather
   than empirically, which is the stronger answer for the purpose it serves: deciding what
   the collector is allowed to treat as a rotation backup.

### Phase III (after Phase II validation)

Implementation begins once Phase II has validated the behaviour and resolved the key
design risks: specifically, that the loss reproduces as described, and that segment
identity across renames has a workable answer. Waiting for that is a sequencing decision
based on evidence, not a hold on anyone else's reply.

Maintainer feedback remains pending and will be incorporated into the design if it
arrives; if it changes the mechanism, I will update this document and say so plainly
rather than quietly reframing what I already wrote. If nothing arrives by the time Phase
II is complete, I will follow up on the issue with the reproduction results attached,
which is a better prompt for a reply than a second ping. I will not treat a lack of
response as agreement with my design.

---

## Resources Used

- KubeRay issue #4830: https://github.com/ray-project/kuberay/issues/4830
- KubeRay issue #4825: https://github.com/ray-project/kuberay/issues/4825
- KubeRay PR #4983: https://github.com/ray-project/kuberay/pull/4983
- KubeRay repository: https://github.com/ray-project/kuberay
- KubeRay `CONTRIBUTING.md`: https://github.com/ray-project/kuberay/blob/master/CONTRIBUTING.md
- KubeRay History Server development guide: https://github.com/ray-project/kuberay/blob/master/historyserver/DEVELOPMENT.md
- Ray log rotation and logging configuration: https://docs.ray.io/en/latest/ray-observability/user-guides/configure-logging.html#log-rotation
- Ray log persistence on Kubernetes: https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/persist-kuberay-custom-resource-logs.html
- Go `fsnotify`: https://pkg.go.dev/github.com/fsnotify/fsnotify
- Go `path/filepath` (`WalkDir`, `EvalSymlinks`, `Rel`): https://pkg.go.dev/path/filepath
- Kubernetes sidecar containers and shared volumes: https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/
