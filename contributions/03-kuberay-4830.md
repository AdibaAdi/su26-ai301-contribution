# Contribution 3: Collect rotated Ray logs in the KubeRay History Server collector

**Contribution Number:** 3
**Student:** Adiba Akter
**Project:** KubeRay (`ray-project/kuberay`)
**Repository:** https://github.com/ray-project/kuberay
**Issue:** [#4830: \[Feature\] Collect rotated logs](https://github.com/ray-project/kuberay/issues/4830)
**Fork:** https://github.com/AdibaAdi/kuberay
**Contribution repo (public):** https://github.com/AdibaAdi/su26-ai301-contribution
**Working branch:** [`feat/4830-collect-rotated-logs`](https://github.com/AdibaAdi/kuberay/tree/feat/4830-collect-rotated-logs)
**Pull request:** [ray-project/kuberay#5101](https://github.com/ray-project/kuberay/pull/5101) — "History Server: collect rotated Ray logs before overwrite"
**Status:** Phase I complete · Phase II complete · Phase III complete · Phase IV complete — PR #5101 open, awaiting maintainer review

---

## Contribution Summary

This contribution targets KubeRay issue #4830: the History Server log collector does not
capture Ray's *rotated* log files, so log content that rotates away while a cluster is
still running is lost before it ever reaches object storage.

The implementation is complete and submitted as
[PR #5101](https://github.com/ray-project/kuberay/pull/5101), which closes #4830. Eight
commits on `feat/4830-collect-rotated-logs` add a rotated-log subsystem beside the existing
collector paths: it discovers completed rotation backups while the pod is running, pins each
one with a hard link before Ray can reuse the filename, uploads it under a unique key, and
releases the link once the bytes are in object storage and Ray has let go of its own. The
pre-existing shutdown walk and `prev-logs` recovery are unchanged.

Validation covers both levels. At the package level there are 179 test functions across six
new test files, passing under the race detector. End to end, a native arm64 Kind cluster with
the KubeRay operator, Ray 2.52.0 and MinIO showed 25 rotated segments recovered per node from
a workload whose local backup ring had already overwritten 20 of them — content that the
unmodified collector would have lost.

This document is written in phase order and preserves what was known at each point. The
Phase I sections keep the findings I labelled preliminary at the time; Phase II records what
I confirmed, what I had to correct, and how the design changed; Phase III records the
implementation and its testing; Phase IV records the pull request, the verification evidence
and what remains open.

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

At the time of writing in Phase I the branch existed only locally and was identical to
upstream `master`, with nothing on it to push. It now carries the eight implementation
commits listed in Phase III and is pushed to the fork.

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

**State of the conversation as of 7 August 2026:** no reply to the design comment has been
posted, and no maintainer has commented on PR #5101. To be explicit about what that does and
does not mean:

- The issue is **not** assigned to me, and I have not asked for it to be.
- Nobody approved the proposed design. The absence of a reply is not agreement, and I did
  not treat it as such at any point.
- The only thing endorsed is the general idea of collecting rotated logs, in the earlier
  project-member comment (link 1) and in the issue author's own framing.
- I proceeded with reproduction and implementation rather than waiting, because reproducing
  the loss and characterising the rotation behaviour is useful evidence whichever mechanism
  is eventually chosen, and because a working implementation with measurements attached is a
  better basis for review than a proposal alone.
- The mechanism changed substantially between that design comment and the submitted code —
  from a filename-based periodic scanner to hard-link pinning with event-driven discovery.
  The "Proposed Direction" section below is preserved as written so the difference is
  visible; the Implementation section explains why it changed. Any feedback that arrives on
  either the issue or the PR will be answered and recorded in the review log at the end of
  this document.

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
- [ ] Feedback on the design: none received; the design was later revised on engineering
      grounds and the change is documented in Phase II and Phase III
- [ ] Issue assignment: not requested, not granted

---

## Phase II: Investigation, Reproduction and Solution Planning

### Development Environment

All Phase II work happens on
[`feat/4830-collect-rotated-logs`](https://github.com/AdibaAdi/kuberay/tree/feat/4830-collect-rotated-logs)
in my fork, `AdibaAdi/kuberay`, which is pushed and tracked as
`origin/feat/4830-collect-rotated-logs`. The branch was created from upstream `master` and has
been rebased more than once as upstream moved; it was rebased onto `dc3afe89` (2026-07-31)
before the pull request was opened, which is the base the submitted commits sit on. Rebasing
was treated as part of the plan rather than an afterthought, and the section on what changed
underneath me explains why that matters more than usual in this subtree.

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
retrieval path. That end-to-end confirmation was done later, in Phase III, following the
walkthrough in `historyserver/DEVELOPMENT.md` (kind, the KubeRay operator via Helm,
`make -C historyserver localimage-build`, `kind load docker-image` for `collector:v0.1.0`
and `historyserver:v0.1.0`, then `historyserver/config/minio.yaml`); the results are in the
"Manual Kind and MinIO integration" section. At the time this Phase II section was written
it had not been run, and the package-level reasoning below stands on its own.

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

In the event I did not do this: the revised mechanism was explained in the pull request
description and in this document, but no follow-up comment was posted on issue #4830 between
the original design and the pull request. That gap is mine, and it is revisited under "What I
Would Do Differently".

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

End-to-end validation was planned here and carried out in Phase III. The shape of it is to
bring up the cluster, operator, MinIO and the data-generating `RayCluster`, lower
`RAY_ROTATION_MAX_BYTES` and `RAY_ROTATION_BACKUP_COUNT` on the Ray containers so segments
rotate in seconds, generate enough log volume to cycle through several rotations while the
cluster stays up, and then confirm two things that no unit test can: that the captured
segments are present in the `ray-historyserver` bucket for a session that never ended, and
that they are visible through the History Server's node-log listing rather than merely
present in storage. Both were confirmed; see "Manual Kind and MinIO integration".

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

## Phase III: Implementation

### Implementation Progress

Implementation lives on
[`feat/4830-collect-rotated-logs`](https://github.com/AdibaAdi/kuberay/tree/feat/4830-collect-rotated-logs)
in my fork and is pushed to `origin`. Eight commits sit on top of upstream `dc3afe89`, and
they were deliberately shaped as tranches that each stand on their own: every one compiles,
is covered by its own tests, and adds a layer the next one builds on. Reviewing them in order
is easier than reviewing the whole subsystem at once, and it also meant that when a later
tranche found a flaw in an earlier assumption, the correction landed as a visible change
rather than being quietly folded into an unreviewable diff.

```
c2ba7533  2026-07-31  [History Server] Add rotated log naming and staging paths
76c3b782  2026-07-31  [History Server] Add rotated log capture state
a72d8578  2026-07-31  [History Server] Add rotated log hard-link capture
11e7dd40  2026-07-31  [History Server] Add rotated log discovery loop
f0a5d424  2026-07-31  [History Server] Add rotated log upload lifecycle
129e0102  2026-08-02  [History Server] Wire rotated log collection into runtime
2d45c6f0  2026-08-06  [History Server] Bound rotated log staging retention
5b8db49a  2026-08-07  [History Server] Keep resolved session path on fallback
```

The hashes changed once since Phase III began: the branch was rebased from `d36899dd` onto
`dc3afe89` before the pull request was opened, so earlier drafts of this document cited the
pre-rebase hashes. The eight above are the ones on the submitted branch; the last was pushed
after the pull request was opened, in response to the automated review recorded in Phase IV.

**`c2ba7533` — [History Server] Add rotated log naming and staging paths** introduces
`rotated_names.go` (352 lines) with `rotated_names_test.go` (486 lines) and adds eight
lines to `historyserver/pkg/utils/constant.go`. It answers the two questions everything
else depends on: what counts as a rotation backup, and where a captured one lives. Ray's
convention is encoded once, in `rotationBackupRe`, and read through `parseBackupName`,
with `evaluateCandidate` deciding eligibility from the entry's mode and whether a known
active base file exists in the same directory — the check that keeps unrelated files
ending in `.<digits>` out of the capture path. Identity comes from `captureIDGenerator`,
which mints a zero-padded nanosecond timestamp plus 64 bits of `crypto/rand` entropy, and
a failure of the entropy source aborts the capture rather than falling back to something
predictable. `captureFileName` and `parseCaptureFileName` attach and recover that ID
around the `.rotated.` separator, so a stored object still carries the operator-readable
rotation name. `clusterIdentity.objectKey` places the result under `clusterlogs.LogsDir`,
and `stagedEntry`, `newStagedEntry`, `parseStagedPath`, `validatePathSegment`,
`validateRelDir` and `relDirFor` validate every component that ends up in a filesystem or
object path, because each of those fields is joined into a path and an unvalidated one is
a traversal. The staging layout itself is the design's quiet centrepiece:
`<staging root>/<session>/<node>/<state>/<relative dir>/<name>.rotated.<capture ID>`,
where `state` is the literal directory name `pending` or `uploaded`. Progress is durable
because it is a directory level, not a field in memory, and
`utils.GetRayRotatedStagingPath` puts that root at `rotated-staging` under `RAY_TMP_ROOT`,
beside `prev-logs` rather than inside it, since the `prev-logs` walker uploads and then
deletes whole node directories.

**`76c3b782` — [History Server] Add rotated log capture state** adds `rotated_state.go`
(145 lines) and `rotated_state_test.go` (226 lines). This is the in-memory half of
identity: `inodeKey` pairs device and inode, `capture` binds that key to its staged entry,
and `capture.releasable` encodes the two-part release condition — uploaded, and a link
count of one. `captureIndex` holds every pinned segment for one collector and is
deliberately lock-free, because a single owner goroutine is its only caller; `add` is
idempotent per inode, which is how a `.1` event and a later `.2` event for the same
physical file collapse into one capture rather than two object keys. `restore` is the
restart path and is stricter than `add`: an inode found under two staging records is a
corrupt tree and is refused, rather than letting the filesystem walk order decide which
record wins. `observeBase` and `baseObserved` remember active log file names per
directory, which is what lets a backup still be recognised during the instant a rotation
cascade leaves the active name unlinked. `validTransition` allows exactly one durable
state change, `pending` to `uploaded`, and nothing moves back.

**`a72d8578` — [History Server] Add rotated log hard-link capture** adds `rotated_fs.go`
(178 lines), `rotated_fs_test.go` (765 lines), and the platform split
`rotated_stat_unix.go` (26 lines) and `rotated_stat_other.go` (15 lines). `captureLink`
is the operation the whole design rests on: a hard link into staging, which costs no copy
and no additional blocks, and after which the segment survives both the rename and the
unlink. `promoteCapture` performs the `pending` to `uploaded` move by doing every fallible
step — the directory creation — before the rename, so that once the atomic rename succeeds
all that remains is an assignment on the owner goroutine and disk and index cannot end up
disagreeing. `releaseCapture` re-reads the link count itself immediately before unlinking
rather than trusting a count passed in, because a caller-supplied count is a claim about
the past, and it removes the index entry only after the unlink, since the kernel may hand
that inode number to an unrelated file the moment the last link disappears. The error
classifiers `isUnsupportedLinkError` (`EXDEV`, `EPERM`, `EACCES`), `isVanished` and
`isAlreadyStaged` separate a deployment that cannot support capture from a race lost to
rotation from an outright bug.

**`11e7dd40` — [History Server] Add rotated log discovery loop** adds
`rotated_collector.go` (712 lines) and `rotated_collector_test.go` (1579 lines). This is
where discovery becomes continuous. `rotatedCollector` runs one goroutine that owns all
state; `fsnotify` watches are installed recursively over the session's `logs/` tree by
`installWatchesRecursive`, and `defaultReconcileInterval` (30 seconds) drives a backstop
sweep, because a watch added to a directory cannot see what was already in it and
`fsnotify` can drop events under load — an `ErrEventOverflow` triggers an immediate
reconcile rather than waiting for the next tick. Startup order is load-bearing and
documented as such: watches first, so no segment can appear and vanish unobserved while
reconstruction runs; then `reconstructStaging`, so a restart adopts what the previous run
pinned instead of minting a second capture ID for it; then the full scan. `inspectFile`
records active base names and routes eligible backups into `capture`, which reads the
inode back from the link it created rather than from the source path — between validating
a candidate and linking it, `raylet.out.1` can already name a different file, and pinning
one inode while recording another would break deduplication, restart recovery and release
at once. `reconstructStaging`, `collectStagedRecords` and `removeSurplusLink` make restart
deterministic: records are sorted into a total order (pending before uploaded, then
capture ID, then path) so nothing depends on walk order, a surplus link is removed only
after confirming it still holds the inode it was recorded with, and an unreadable staging
subtree stops startup rather than leaving links pinning inodes that nothing owns.

**`f0a5d424` — [History Server] Add rotated log upload lifecycle** adds
`rotated_uploader.go` (1090 lines) and `rotated_uploader_test.go` (2097 lines), and
revises `rotated_collector.go` and `rotated_names.go` in the process. Uploading is pushed
onto a worker and never performed on the owner goroutine, because a `WriteFile` sitting
between a rotation and its capture would lose exactly the segments the collector exists to
keep. `uploadWorker.execute` validates on the open descriptor rather than the path — an
`fstat` after `open` answers "what did I actually open?" — and then hands the `*os.File`
straight to `storage.StorageWriter.WriteFile`, so a large segment streams rather than
being read into the sidecar's memory. Results come back as immutable values and are
matched against a comparable `uploadIdentity` before anything is applied, which is how a
result that outlived its capture is discarded instead of mutating whatever now occupies
the same inode. Failures follow `defaultUploadBackoff`, a fixed 1s-to-5m sequence without
jitter, since one collector uploads one capture at a time and a fixed sequence is exactly
assertable in tests; a failed upload leaves the capture pending with its capture ID,
staged link and object key untouched, so the retry is the identical write to the identical
key. Promotion happens only after the write returns, and if the rename itself fails the
capture stays consistently pending and only the promotion is retried — re-uploading would
be a second write of bytes the store already has. Release then runs through
`releaseIfUnheld`, which treats a remaining link as the expected steady state rather than
a failure, since Ray keeps a rotated segment until its backup count rolls it off.

Backpressure arrived in the same tranche, through `stagedBytes` and `intakeGate`. Staging
is bounded by a high-water and low-water pair, validated by `validateWatermarks` so that a
configuration which could never resume, or would resume the instant it paused, is rejected
at construction. The gate pauses *intake only*: nothing already captured is ever evicted,
because deleting pending data to accept newer pending data would trade one loss for
another and lose the older segment for certain. `ENOSPC` from `os.Link` is handled
separately from the watermarks, since another process can fill the volume regardless of
what the collector is holding, and it is cleared only by evidence — a release that freed
blocks, or a probe link that succeeded — never by elapsed time. Byte accounting can also
declare itself untrusted through `markStale` and `recompute`; an untrusted total may still
pause intake, erring towards holding back, but may never resume it.

**`129e0102` — [History Server] Wire rotated log collection into runtime** adds
`rotated_runtime.go` (891 lines) with `rotated_runtime_test.go`, and reworks `collector.go`.
`rotatedSupervisor` owns at most one collector at a time and binds it to one session on one
node, because both values are baked into every staging path and every object key; a change to
either half retires the old collector completely — final reconciliation, bounded drain, stop,
goroutine join — before the replacement is constructed, so the staging root never has two
owners. `preflight` rejects, once per identity, the deployment conditions the collector would
otherwise discover one segment at a time: a missing logs directory, a staging root it cannot
write to or clean up, a platform without inode identity, and a staging root on a different
filesystem from the logs, where a hard link cannot reach.

In `collector.go` the changes are in three places. `Run` starts rotated collection before the
existing goroutines, since a segment rotated away in the first seconds of a session is already
gone by the time the shutdown walk runs. `shutdownLogCollection` replaces the direct call to
`processSessionLatestLogs` with an ordered sequence: close session-transition admission and
wait for one already in flight, retire the rotated subsystem with a bounded drain, then run
the existing walk from an immutable snapshot of what the live session meant when shutdown
began. And `PollActiveSessionChanges` was restructured around an explicit `sessionTransition`
so that each outgoing session is relocated under the node it actually ran on, rather than
under whichever node ID happened to be current — the previous code passed a single node ID to
`MoveLeftoverSessionLogs`, which stamps every inactive session directory with it.

**`2d45c6f0` — [History Server] Bound rotated log staging retention** came out of reviewing
the branch as a whole rather than from a failing test. A capture is a
hard link, so while uploads are failing the collector keeps inodes alive that Ray has already
released, and nothing bounded that. The commit adds a second byte total: `StagedBytes` stays
what it was, the logical size of everything pinned, and `RetainedBytes` counts only captures
where the collector holds the last link (`nlink == 1`) — the bytes that exist because of this
feature. Backpressure is driven by the second. The distinction matters because a capture Ray
still has its own link to shares Ray's blocks and costs nothing extra, so gating on the
logical total would have charged the collector for Ray's entire backup ring and paused
capture during ordinary healthy rotation. Link counts come from stats the code already
performed and previously discarded, are rebuilt from the filesystem on restart, and are
refreshed in the existing reconcile sweep, since Ray unlinking its own name changes nothing
about the staging path and produces no event.

**`5b8db49a` — [History Server] Keep resolved session path on fallback** is the last commit
and answers the automated review finding recorded in Phase IV. `PollActiveSessionChanges`
seeds its transition state with the symlink-resolved form of the configured `SessionDir`,
because every later observation is a resolved symlink target and `advanceSession` compares the
two by string. The fallback taken when `session_latest` cannot be resolved used the raw
configured value instead, so two spellings of one directory compared unequal and the still-
running session was treated as an outgoing one: its `logs/` directory was queued for
relocation into `prev-logs` and its verified node identity was cleared. The two spellings are
reachable because `utils.GetSessionDir` falls back to `os.Readlink`, which resolves only the
`session_latest` link and leaves any symlinked component above it in place. The correction is
to fall back to the already-resolved `startupDir`, so the fallback value is always the string
`st.dir` holds; no path canonicalisation was introduced anywhere else. Nothing is suppressed
by declining to act, since `SessionDir` names the session the poller started with and can
never be evidence of a new one, and the first tick that resolves `session_latest` observes any
real change. `TestPollActiveSessionChangesFallbackKeepsTheResolvedSessionPath` is the
regression test.

### Key files changed

| Path | Lines | Role |
|---|---:|---|
| `rotated_names.go` | 361 | Rotation-name parsing, capture IDs, object keys, staged-path validation |
| `rotated_state.go` | 145 | `inodeKey`, `capture`, `captureIndex`, release predicate, base-name memory |
| `rotated_fs.go` | 192 | `captureLink`, `promoteCapture`, `releaseCapture`, error classifiers |
| `rotated_stat_unix.go` / `rotated_stat_other.go` | 26 / 15 | `inodeFromFileInfo` behind a build constraint |
| `rotated_collector.go` | 992 | Owner goroutine, fsnotify watches, reconcile sweep, capture, staging reconstruction |
| `rotated_uploader.go` | 1155 | Upload worker, retry scheduler, byte accounting, intake gate |
| `rotated_runtime.go` | 891 | `rotatedSupervisor`, preflight, failure classification, bounded drain |
| `collector.go` | +623 / −67 | Subsystem start-up and shutdown ordering, session-transition rework |
| `pkg/utils/constant.go` | +8 | `GetRayRotatedStagingPath` |

Test files add 8,383 lines across `rotated_names_test.go`, `rotated_state_test.go`,
`rotated_fs_test.go`, `rotated_collector_test.go`, `rotated_uploader_test.go` and
`rotated_runtime_test.go`. The whole change is 16 files against `upstream/master`, and it
touches no package outside `historyserver/pkg/collector/logcollector/runtime/logcollector`
except the one path helper in `pkg/utils/constant.go`.

### Testing and Validation

The branch carries 179 test functions across six files — 13 in `rotated_names_test.go`, 9 in
`rotated_state_test.go`, 18 in `rotated_fs_test.go`, 34 in `rotated_collector_test.go`, 41 in
`rotated_uploader_test.go` and 64 in `rotated_runtime_test.go`. They are written to exercise
the real filesystem rather than a model of it: real files, real hard links, real link counts,
with only the clock, the ticker, the watcher, the storage writer and the link call itself made
injectable so that races can be driven deterministically instead of waited on. They follow the
package's existing conventions — the in-memory `storage.StorageWriter` fake and temp-root
fixture patterns established by `collector_test.go` — rather than introducing a parallel
harness.

The naming and state tests pin down identity. `TestCaptureIDSurvivesRestart` and
`TestCaptureIDFormatAndUniqueness` check that IDs remain unique across a simulated
restart, which is the property that stops a restarted collector minting an ID that an
earlier object already used; `TestObjectKeyIsFlatAndOwnerAware` checks that keys stay flat
beside a node's other logs and that RayJob and RayService clusters keep nesting under the
owner name; `TestNewStagedEntryRejectsUnsafeComponents` and
`TestParseStagedPathRejectsMalformed` cover the path-validation surface;
`TestCaptureIndexAddIsIdempotentPerInode` and
`TestCaptureIndexDistinctInodesAreDistinctCaptures` are the two halves of the
rename-versus-reuse question.

The filesystem tests establish that capture actually defeats rotation.
`TestCaptureSurvivesRotationAndDeletion` links a segment, renames the original from `.1`
to `.2`, confirms the staged link still reports the same inode with a link count of two,
deletes the rotated file, and confirms the inode is unchanged, the count has fallen to
one, and the content is readable in full. `TestCaptureOfSuccessiveSegmentsAtSamePath`
proves the inverse: two generations at the same rotation filename report different inodes,
because the first link is still holding the old one alive, and therefore receive distinct
capture IDs and distinct object keys.
`TestReleaseCaptureRequiresUploadedAndLastLink`, `TestReleaseCaptureRefusesWhenInodeChanged`
and `TestPromoteCaptureFailureLeavesDiskAndIndexPending` cover the state machine's
refusals rather than its happy path.

The discovery tests target the ugly cases. `TestCollectorWatchesTreeBeforeReconstruction`
holds the collector inside the reconstruction window and proves the tree is already
covered; `TestCollectorPeriodicReconciliationCatchesMissedEvents` and
`TestCollectorReconcilesImmediatelyAfterOverflow` cover the two ways events go missing;
`TestCollectorPinsTheInodeItActuallyLinked` and
`TestCollectorCapturesGenerationThatReplacedACapturedOne` cover the rename race that
motivated reading the inode back from the link; `TestCollectorStagingConflictResolutionIsDeterministic`
and `TestCollectorFailsWhenStagingCannotBeRead` cover restart reconstruction, including
the cases where startup is supposed to fail rather than proceed; and
`TestCollectorStateIsOnlyTouchedByTheOwnerLoop` is a structural assertion about the
concurrency model itself.

The upload tests cover the lifecycle end to end.
`TestUploadRetriesFollowBackoffAndReuseTheSameKey` asserts the retry schedule and that
every attempt writes the same key; `TestRemoteSuccessWithLocalPromotionFailureRetriesPromotionOnly`
covers the case where the bytes are safe but the local rename failed;
`TestRestartDoesNotReuploadUploadedEntries` and
`TestRestartReleasesReconstructedUploadedEntries` cover adoption after a restart;
`TestUploadedCaptureIsReleasedOnlyAfterRayLetsGo` covers the release condition;
`TestBackpressurePausesIntakeWithoutEviction`, `TestHighWaterStopsCaptureWithinOneScan`,
`TestLaterENOSPCIsNotClearedByAnEarlierSuccess`, `TestENOSPCRecoversWhenSpaceIsFreedExternally`
and `TestCapacityProbeDoesNotBypassTheWatermark` cover backpressure in both its forms;
`TestLocalValidationFailureStopsTheCollector` and
`TestUploadResultFailurePolicySeparatesLocalFromTransport` cover the deliberate decision
to fail closed on a local contradiction; and
`TestWorkerDoesNotStartTheRemoteWriteAfterQuit`, `TestLateUploadResultAfterStopCannotMutateState`
and `TestUploaderStopsWithoutLeakingGoroutines` cover shutdown.

The retention accounting added in `2d45c6f0` has its own regression coverage, because the
mistake it corrects is one that unit tests could easily have missed.
`TestRetainedBytesExcludeBlocksRayStillOwns` captures a 500-byte segment while Ray's link is
intact against a 10-byte watermark and asserts `StagedBytes` is 500, `RetainedBytes` is zero,
and intake stays open — reverting the gate to the logical total makes this test fail, which is
what makes it a regression test rather than a passing one.
`TestRetainedBytesCountCapturesRayHasRolledOff` removes only Ray's link and asserts that a
reconcile sweep moves the same 500 bytes into `RetainedBytes`, which is the transition no
filesystem event announces. `TestOutageRetainsBoundedDiskThenResumes` drives the whole outage
sequence: writes fail, retained bytes cross the high mark, intake pauses, nothing already
captured is evicted, retries keep running, and once writes succeed the sole links release and
intake reopens. `TestReconstructionComputesRetainedFromLinkCounts` starts a collector over a
staging tree holding one shared and one sole-owned capture and asserts the split is rebuilt
from link counts alone. `TestRuntimeConfiguresBoundedIntake` is the wiring test: it asserts
that the supervisor-built production configuration actually carries a positive high-water
mark with a lower low-water mark, which is the gap that let an earlier revision ship with
backpressure implemented but switched off.

Validation runs from the `historyserver/` module directory. The full collector suite passes
under the race detector; `go vet` is clean across the module; the module cross-compiles for
the deployment targets and for a non-Unix target, which is what exercises the
`rotated_stat_other.go` stub; and `golangci-lint` with the repository's own root
configuration reports no findings in any `rotated_*` file. The eight findings it does report
are all in pre-existing upstream files — `collector.go`, `collector_test.go` and
`endpoint_fetch_once.go` — and I confirmed each one exists in `upstream/master` rather than
being something this work introduced. The pre-existing collector tests continue to pass under
the race detector as well, which matters because the argument for this change is that it adds
a path rather than altering one.

```bash
cd historyserver
go test -race -count=1 ./pkg/collector/logcollector/runtime/logcollector
go test -race -count=1 ./pkg/collector/...
go test -count=1 ./pkg/storage/...
go test -count=1 ./pkg/utils/...
go vet ./...
go build ./...
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build ./...
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build ./...
golangci-lint run -c ../.golangci.yml ./pkg/collector/logcollector/...
```

The rotated-log package was also run repeatedly under the race detector rather than once,
including in-process repeats with `-count=3` to surface state leaking between runs. During
that stress one pre-existing test, `TestRuntimeConcurrentSessionChangesKeepOneOwner`, was
found to depend on goroutine scheduling: it called `ensureRotatedCollection` directly and
asserted immediately, without the round trip through the owner goroutine that the harness uses
elsewhere, so a timing change in an unrelated commit could break it. It now synchronises the
same way the rest of the harness does. Earlier in the branch's life the subsystem was also put
through a mutation-testing pass and a `git diff --check` and clean-apply check against the
original upstream baseline.

### Manual Kind and MinIO integration

Package tests cannot show that a captured segment reaches a bucket, so the flow was run end to
end on a local cluster: `kind`, the KubeRay operator installed via Helm, `rayproject/ray:2.52.0`,
MinIO for object storage, and the collector and History Server images built from this branch.
The run was performed manually rather than as an automated e2e suite, and it was performed at
commit `129e0102`; the two commits added afterwards, `2d45c6f0` and `5b8db49a`, are covered by
the package and race tests rather than by this cluster run.

The workload is deliberately deterministic rather than "generate some logs". Rotation was
configured with `RAY_ROTATION_MAX_BYTES=65536` and `RAY_ROTATION_BACKUP_COUNT=5`, and one Ray
task was hard-pinned to the head node and one to the worker, each emitting 6,400 uniquely
marked 256-byte lines carrying a run-unique marker, the node role, and a sequence number. That
gives 1,638,400 bytes per node against a 65,536-byte rotation threshold: 25 rotations per node,
against a backup ring five deep. The arithmetic is the point — 20 of those 25 rotations per node
must have been overwritten locally during the run.

Observed:

- 25 marked rotated objects were uploaded for the head node and 25 for the worker.
- All 50 landed under the correct session and Ray node prefixes, with no head/worker
  cross-attribution.
- MinIO `lastModified` timestamps for every object fall between 02:30:25Z and 02:30:50Z, while
  both Ray pods were still `Running` and `Ready`; the objects were listed and downloaded at
  02:54:34Z with the pods still up. Collection happened during the run, not at shutdown.
- Ray and collector restart counts stayed at zero throughout.
- MinIO held marked sequences 1 through 6,399 for each node.
- The live pod filesystems held only sequences 5,120 through 6,400 — the active file plus five
  backups. Sequences 1 through 5,119 were gone from both pods.

That last pair is the result the issue asks for: 20 of the 25 rotations per node had already
been overwritten on the node and were recoverable only from object storage.

### Archived History Server verification

Storage alone is not the deliverable; the History Server has to be able to serve the content.
After deleting the `RayCluster`:

- All 50 marked objects remained in MinIO.
- The History Server discovered the archived session and served it through `/enter_cluster`.
- The archived node-log listing returned all 25 rotated filenames for the head node and all 25
  for the worker.
- One marked head object and one marked worker object were downloaded through the History
  Server's log-download route, with the correct marker, the correct role and the correct
  sequence range in each.
- An ordinary non-rotated `raylet.out` was listed and downloaded through the same route,
  confirming the pre-existing path still works alongside the new one.

Two qualifications belong with that result. The History Server's download handler is
line-oriented and omits the final trailing newline, so a downloaded object is one byte shorter
than the stored object; every other byte matched, and the behaviour is pre-existing and
unrelated to rotated capture. And a collector-side `.rotated.*` filename returns not-found on
the *live* log route, because that route proxies Ray's dashboard, which only knows Ray's own
local filenames; the archived object-storage route is the one that lists and serves these
objects, and it does.

### Challenges and Engineering Decisions

Several things only became visible once the code existed, and each of them changed the
design rather than being worked around.

The first was that the source path cannot be trusted as the thing being captured. The
obvious implementation reads the inode of `raylet.out.1`, links it, and records what it
read. But rotation reuses that filename constantly, so between the read and the link the
name can already point at a different file, and the staging link would then pin one inode
while the index recorded another — breaking deduplication, restart recovery and release
simultaneously. The correction was to read the inode back from the link that was actually
created and register only that value, with the source inode kept for diagnostics alone.
For the same reason there is deliberately no dedup shortcut before linking: returning
early because the source's current inode looks familiar could skip a generation that had
just replaced it.

The second surfaced when backpressure was added in `f0a5d424`, and it changed a function
signature that had looked settled: `pinnedInode` was extended to return the file's size
alongside its identity, so that byte accounting describes exactly the file that was
pinned rather than whatever the source path happens to hold a moment later. The same
tranche revealed that a disk-full pause, implemented naively, latches permanently — the
capture path returns before ever attempting a link, so if another process filled the
volume and this collector holds nothing it can release, nothing would ever discover that
space came back. The fix was a single capacity probe per maintenance sweep, using a real
eligible backup so that a successful attempt preserves data instead of merely testing the
filesystem, and `armProbe` refuses to probe a watermark pause, since that one is the
collector's own limit rather than the filesystem's.

The third was that retryability cannot be inferred from an error value. A logs directory
that does not exist yet and a durable staging link that has disappeared both surface as
`fs.ErrNotExist`, and they are opposites: the first is the session poller running ahead of
Ray's own directory creation, the second is the staging volume contradicting the index and
is fatal by design. Classifying on the errno would restart the second every fifteen
seconds forever. The design was corrected so that nothing is retryable unless it is
explicitly marked at the point of failure, by `retryableRotated`, and everything else is
durable by default. The one place that marks a failure retryable is watch installation,
which can fail because inotify or descriptor limits are momentarily exhausted —
`isWatchResourceExhausted` covers `ENOSPC`, `EMFILE` and `ENFILE` — and that classifier is
consulted only for watches, never for captures, where `ENOSPC` means something entirely
different.

The fourth is the one I cannot engineer away, and it comes from the storage interface
itself: `storage.StorageWriter.WriteFile` takes no `context.Context`, so an upload that
has entered the object client cannot be canceled. Shutdown therefore has to be a bounded
budget rather than a wait. The worker re-checks a quit channel after receiving a job and
again after local validation, so an upload that has not started never starts, and only a
write already under way outlives the owner; when that happens its result is discarded and
the capture stays pending for the next run. The drain budget is deliberately small — a few
seconds — because everything it spends is taken from the legacy shutdown walk that runs
after it, and that walk is the pre-existing data path for every file still in the live
tree. Kubernetes' default 30-second termination grace period is the ceiling both halves
share, and KubeRay does not raise it for the collector sidecar.

Some decisions went past the minimum a working implementation would need, and they were
deliberate. Restart reconstruction was built rather than accepting duplicate uploads after
a restart, because the alternative silently doubles storage for every long-lived cluster.
Reconstruction is deterministic rather than merely functional: records are put into a
total order before adoption, so two runs over the same staging tree resolve a conflict
identically instead of depending on filesystem walk order. Nothing captured is ever
silently evicted — backpressure stops intake and says so, but never deletes data the
collector already holds, and the honest cost of that policy is written into the code
comment rather than left implicit. And failure handling is conservative throughout: an
unreadable staging subtree stops startup rather than starting without owning links that
pin inodes; a local contradiction between the index and the volume fails closed rather
than retrying forever on a transport schedule; and a surplus staging link is removed only
after confirming it still holds the inode it was recorded with, so a path that has since
become something else is reported and left alone.

The fifth challenge came from the integration environment rather than the code. My first
attempt built the whole cluster as `linux/amd64` under emulation on an arm64 host. `kubeadm`
got as far as writing the static pod manifests, but no container was ever created, and the
kubelet logged `failed to generate seccomp spec opts: seccomp is not supported` 4,296 times.
Reading `/proc/self/status` inside the emulated node confirmed `Seccomp: 0` — the emulated
runtime does not expose seccomp at all, so containerd cannot construct a sandbox spec. That is
an environment limitation rather than anything to do with this change, and the fix was to stop
fighting it: rebuild the cluster natively on arm64 with an arm64 node image and arm64 builds of
the operator, collector, History Server, Ray and MinIO. Everything after that point ran on the
host's own architecture.

The sixth is the one that survived longest without being noticed, and it was found by reviewing
the branch rather than by any failing test. Backpressure was measured against the logical size
of everything staged. But a capture is a hard link, so while Ray still holds its own link to a
rotated segment the blocks are shared and the collector is costing nothing extra; those bytes
only become the collector's responsibility once Ray rolls the segment off its ring and the
staging link is the last reference. Charging for the shared case meant the watermark tracked
Ray's entire backup ring, which at Ray's default 512 MiB rotation size is roughly 2.5 GiB for a
single log file — enough to pause capture permanently during completely healthy operation and
silently stop the feature doing its job. The correction was to keep the logical figure as a
diagnostic and add `RetainedBytes`, counting only captures with `nlink == 1`, and to drive
backpressure from that. What made it a small change rather than a redesign is that the link
count was already being read at every accounting site and thrown away.

---

## Phase IV: Pull Request and Verification

### Pull request

| | |
|---|---|
| Pull request | [ray-project/kuberay#5101](https://github.com/ray-project/kuberay/pull/5101) |
| Title | History Server: collect rotated Ray logs before overwrite |
| Closes | [#4830](https://github.com/ray-project/kuberay/issues/4830) |
| Base | `ray-project/kuberay` `master` |
| Head | `AdibaAdi/kuberay` `feat/4830-collect-rotated-logs` |
| Head commit | `5b8db49a` (8 commits ahead of `upstream/master`, 0 behind) |
| Opened | 7 August 2026 |
| Status | Open, awaiting maintainer review; CI in progress on the latest commit |

The description follows KubeRay's pull request template and opens under "Why are these changes
needed?" with the problem rather than the mechanism: Ray rotates logs to bound local disk, the
collector previously relied on collecting the remaining log directory at shutdown or through
`prev-logs` recovery, and for long-running workloads that means an older backup can be
overwritten before it is ever uploaded, leaving part of the history unrecoverable even though
object storage has no such limit. It then covers head and worker support, hard-link
preservation, the `.rotated.<capture-id>` naming that keeps successive generations of a reused
filename distinct, event-driven discovery with periodic reconciliation as a backstop, staging
and restart reconstruction, the upload and retry lifecycle, session and node identity across
transitions, and the retained-byte backpressure with its 1 GiB and 512 MiB watermarks. It
closes #4830, and records that unit tests and manual tests were run, with the validation
commands listed.

### Verification

Each item below was confirmed by a named test, by the Kind and MinIO run, or by both.

- Completed rotation backups are captured while the pod is running, before Ray reuses the
  filename. Package: `TestCaptureSurvivesRotationAndDeletion`,
  `TestCollectorCapturesBeforeSourceIsDeleted`. Cluster: 50 objects uploaded while both pods
  were `Running` and `Ready`.
- Distinct generations that occupy the same rotation filename stay distinct.
  `TestCaptureOfSuccessiveSegmentsAtSamePath`,
  `TestCollectorCapturesGenerationThatReplacedACapturedOne`,
  `TestSuccessiveGenerationsProduceDistinctKeys`. Cluster: 25 distinct objects per node, all
  from the reused `.1` name.
- The head collector is validated: 25 marked objects, correct head node ID.
- The worker collector is validated: 25 marked objects, correct worker node ID.
- Session and node attribution is correct, with no cross-attribution between head and worker
  across all 50 objects. `TestPollActiveSessionChangesFilesEachSessionUnderItsOwnNode`,
  `TestPollActiveSessionChangesNeverFallsBackToThePreviousNode`.
- Duplicate capture and duplicate upload are prevented, by inode identity, by upload state, by
  a single in-flight job, and by matching a returning result against the capture it was
  submitted for. `TestNoDuplicateQueueOrInFlightJobs`,
  `TestCollectorDiscardsSurplusLinkForAnAlreadyCapturedInode`, `TestStaleUploadResultCannotPromote`,
  `TestRestartDoesNotReuploadUploadedEntries`.
- Restart recovery adopts the previous run's staging without minting new identities or
  re-uploading. `TestCollectorReconstructsStagingWithoutNewCaptureIDs`,
  `TestRestartUploadsPendingWithoutMintingNewIDs`.
- The archived History Server lists the captured segments: all 25 head rotations and all 25
  worker rotations.
- The archived History Server downloads them with correct content and attribution, verified for
  one head object and one worker object.
- Ordinary non-rotated collection is unaffected. `TestRotatedAndLegacyObjectKeysAreDisjoint`,
  `TestLegacyShutdownUploadsEveryLiveFileEvenWhenAlreadyCaptured`,
  `TestLegacyShutdownUnchangedWithoutRotatedCollection`. Cluster: `raylet.out` listed and
  downloaded through the same route.
- Staging retention is bounded during a storage outage, without evicting captured data.
  `TestOutageRetainsBoundedDiskThenResumes`, `TestRetainedBytesExcludeBlocksRayStillOwns`,
  `TestRuntimeConfiguresBoundedIntake`.
- A transient `session_latest` resolution failure is not mistaken for a session change: the
  live session's logs stay in place rather than being relocated into `prev-logs`, and the
  verified node identity is unchanged. `TestPollActiveSessionChangesFallbackKeepsTheResolvedSessionPath`.

### Before and after

The clearest evidence is the arithmetic of the deterministic workload rather than any
screenshot. Each node emitted 6,400 marked lines of 256 bytes against a 65,536-byte rotation
threshold with five backups retained, which is 25 rotations per node against a ring that can
hold five.

Before this change, the collector uploads what is on disk when it runs, so the recoverable set
is whatever the ring still holds: sequences 5,120 to 6,400 per node, five backups plus the
active file. Sequences 1 to 5,119 are unlinked by Ray during the run and have no upload path.

After this change, MinIO held all 25 rotations per node, covering sequences 1 to 6,399, while
the pod filesystems still held only 5,120 to 6,400. Twenty of the twenty-five rotations per node
existed only in object storage, and all 50 objects survived deletion of the `RayCluster` and were
listed and downloaded through the History Server afterwards.

### Maintainer Feedback and Review Log

**Human maintainer review.** No maintainer review comments have been received on PR #5101 as of
7 August 2026. The pull request is open, has no assignee and no requested reviewers, and is
awaiting review. Upstream CI re-ran on the latest commit and was still in progress when this was
written, so no CI result is claimed. No response to the original design comment on issue #4830
has been received either.

**Automated review, and the response to it.** One automated review was posted by Cursor Bugbot.
Bugbot is an automated reviewer, not a KubeRay maintainer, and it is recorded separately for that
reason. It raised one medium-severity finding, "Fallback skips resolved session path", against
`historyserver/pkg/collector/logcollector/runtime/logcollector/collector.go`. The sequence it
described was investigated rather than accepted on sight, and it turned out to be a real
lifecycle bug whose consequence was worse than the report suggested:

1. `PollActiveSessionChanges` seeds `sessionTransition.dir` with the symlink-resolved form of the
   configured `SessionDir`, and `advanceSession` compares later observations against it by string.
2. The fallback taken when the initial `session_latest` resolution fails assigned the raw
   `SessionDir` instead, so two spellings of one live directory compared unequal.
3. A focused regression test written against the unmodified branch reproduced the failure and
   showed the damage: the poller logged a session change, **relocated the live session's `logs/`
   directory into `prev-logs` while Ray was still writing to it**, and cleared the verified node
   identity. The rotated collector was left running over a tree that had been moved.
4. The two spellings are reachable in practice because `utils.GetSessionDir` falls back to
   `os.Readlink`, which resolves only the `session_latest` link and leaves any symlinked component
   above it in place.
5. The correction reuses the already-resolved `startupDir` for the fallback, so its value is always
   the string `st.dir` holds. No package-wide path canonicalisation was added. Genuine transitions
   are unaffected: the fallback is reachable only when there is no resolvable evidence of any
   session, and the first tick that resolves `session_latest` observes any real change.
6. The regression test passes after the fix, and the race suite, `go vet` and `go build` were
   re-run clean before the change was committed.
7. The fix was committed as `5b8db49a` and pushed to PR #5101, and a reply on the Bugbot review
   thread confirmed the finding and described the correction.

If review feedback arrives from a maintainer on the issue or the pull request, it will be answered
there and recorded in this section with the commit that responds to it.

### Known limitations

- The cluster validation is a manual Kind and MinIO run, not an automated end-to-end test and not
  a production cloud-storage validation. It exercises MinIO's S3-compatible API rather than S3,
  GCS, Azure Blob or Aliyun OSS.
- That run was performed at `129e0102`. The retention accounting in `2d45c6f0` and the session
  fallback fix in `5b8db49a` are covered by package and race tests only.
- Capture depends on the collector being able to hard-link Ray's files, which requires both on the
  same filesystem and adequate permissions. The default KubeRay manifest satisfies both, since the
  containers share one `emptyDir`; a custom image whose Ray user differs from the collector's UID
  degrades to a warning and a skip rather than a failure.
- Anything still unsent when the bounded shutdown drain expires stays on the staging volume. That
  is recoverable across a collector restart inside a surviving pod, but not across pod deletion,
  because `/tmp/ray` is an `emptyDir` that dies with the pod.
- While intake is paused at the high-water mark, segments Ray rotates away are not captured. That
  is a deliberate trade against unbounded disk retention, not an oversight.
- Rotated objects are served by the archived object-storage route. The live log route proxies
  Ray's dashboard and therefore does not know these filenames.

---

## Learnings and Reflections

### Technical Skills Gained

The most transferable thing I learned is that a hard link is a reference-counting tool, and that
`nlink` is a readable answer to "who is keeping these bytes alive?". The whole design turns on
that: a link is a second reference to an inode, so creating one makes the data survive both the
rename and the unlink, and `nlink == 1` is a precise, checkable statement that the collector is
now the only holder. That single fact decides when a capture is safe to release, when the
collector is responsible for disk, and how a restart rebuilds its state from the filesystem with
nothing persisted.

I also learned to build a single-owner concurrency model rather than reaching for mutexes. All
collector state lives on one goroutine, and every external question — a snapshot, statistics, a
forced reconcile — is a channel round trip. That makes "look up the inode, create the link,
register the capture" indivisible without any locking, and it is what let the slow, uncancelable
part, the object-store write, be pushed onto a worker without letting storage latency delay
discovery. Where locks were unavoidable, in the supervisor, I learned to write the ordering down
as an invariant and keep the lock off every path that waits.

Two smaller skills. Classifying failures by the operation that produced them rather than by the
error value: a missing logs directory and a missing staging link are both `fs.ErrNotExist` and
are opposites, so retryability has to be marked at the point of failure. And pairing event-driven
discovery with reconciliation, since a watch cannot see what a directory already contained and
`fsnotify` drops events under load, so neither mechanism alone is sufficient.

On the Kubernetes side I got a lot more comfortable debugging a cluster from the bottom up: reading
kubelet and containerd logs when no pod was ever created, checking `/proc/self/status` for kernel
capabilities inside a node, and understanding how emulation changes what a container runtime can
actually do.

### Challenges Overcome

The design I posted publicly was not strong enough, and I had to say so. A filename-based periodic
scanner cannot solve this problem: it is not only that fast rotation outruns the scan, it is that
the filename is a reused slot, so the scanner both misses content it has never uploaded and
re-uploads content it already has. Replacing that with hard-link pinning meant abandoning the
approach I had already described in public.

Ray reusing numbered backup paths produced a subtler trap. The obvious implementation reads the
inode of `raylet.out.1`, links it, and records what it read — but rotation can replace that file
between the read and the link, leaving the staging link pinning one inode while the index records
another. The correction was to read the inode back from the link that was actually created, and to
remove the tempting dedup shortcut before linking, since an early return could skip a generation
that had just replaced the one being examined.

Session and node transitions took longer than the capture logic. The pre-existing relocation helper
takes a single node ID and stamps every inactive session directory with it, which files a session's
logs under a node that never wrote them, so each outgoing session had to be tracked with the node it
actually ran on and relocated individually before any broad sweep runs.

The amd64-under-emulation cluster failure cost real time and turned out not to be my code at all;
the fix was to stop trying to make the emulated environment work and rebuild natively on arm64.

The last one was found by re-reading my own work rather than by a test: backpressure was measuring
the wrong thing. Distinguishing bytes the collector has merely pinned from bytes the collector alone
is keeping allocated was the difference between a limit that protects the node and a limit that
would have quietly disabled the feature during normal operation.

The automated review after submission reinforced a habit worth keeping: turning a reviewer's claim
into a failing regression test before editing anything proved the bug was real, showed it was worse
than reported — the live session's logs were actually relocated — and left behind a test that keeps
it fixed.

### What I Would Do Differently

**Identify the disk-retention invariant on day one.** I knew from the beginning that pinning a link
prevents reclamation — I wrote it down in Phase II — but I treated it as a caveat rather than as an
invariant that needed a precise definition. If I had asked "exactly which bytes exist because of
this feature?" before writing the accounting, the `StagedBytes` versus `RetainedBytes` split would
have been the first design rather than the last commit.

**Stand up the native-architecture integration environment before writing the implementation.** I
built the cluster after the code was largely done, and lost a working session to an emulation
limitation that had nothing to do with the change. Bringing up the environment early would have
surfaced that at a point where it cost nothing.

**Design the deterministic proof earlier.** The marker workload — unique run ID, role, sequence
number, sizes chosen so the rotation arithmetic is exact — is what turned "rotated logs are
collected" into "20 of 25 rotations per node existed only in object storage." I built that late.
Designing the evidence first would have given me a sharper target throughout, and would have made
the intermediate cluster runs far more informative than "some objects appeared."

**Raise the mechanism change on the issue when I made it, not after.** I posted a scanner design,
then changed to hard-link pinning on engineering grounds, and the public thread still shows only the
original. The reasoning is sound and it is documented here and in the pull request, but the
maintainers who asked for a design first deserved to see the revision at the point it happened. The
lesson from Contribution 1 was to open a scoping conversation before implementing; the version of
that lesson I still need is to keep the conversation open *while* implementing.

---

## Resources Used

- KubeRay issue #4830: https://github.com/ray-project/kuberay/issues/4830
- KubeRay PR #5101 (this contribution): https://github.com/ray-project/kuberay/pull/5101
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
