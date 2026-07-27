# Contribution 3: Collect rotated Ray logs in the KubeRay History Server collector

**Contribution Number:** 3
**Student:** Adiba Akter
**Project:** KubeRay (`ray-project/kuberay`)
**Repository:** https://github.com/ray-project/kuberay
**Issue:** [#4830 — \[Feature\] Collect rotated logs](https://github.com/ray-project/kuberay/issues/4830)
**Fork:** https://github.com/AdibaAdi/kuberay
**Contribution repo (public):** https://github.com/AdibaAdi/su26-ai301-contribution
**Branch (local, not yet pushed):** `feat/4830-collect-rotated-logs`
**Status:** Phase I complete — design shared publicly on the issue with the author and project members tagged; Phase II investigation beginning. No implementation started, no upstream PR opened.

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

Everything below is Phase I work. Nothing has been implemented, no bug has been
reproduced in a running cluster, no tests have been written or run, and no upstream pull
request exists. The findings in this document come from reading the code and Ray's
documentation, and I have labelled them as preliminary throughout.

---

## Why I Chose This Issue

I picked #4830 because it is the kind of problem I keep running into from the other
side. My background is Python, backend services, and testing, and most of my recent
interest has been in AI infrastructure — the boring, unglamorous layer that decides
whether an ML job is debuggable at 3 a.m. Log collection for a distributed Ray cluster is
exactly that layer. It is not a flashy feature, but if it silently drops data, every
downstream investigation gets harder.

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
Contribution 1 — open a scoping conversation with maintainers *before* planning an
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
run out of disk, but the History Server writes to object storage, which has no such limit
— so there is no good reason to lose the rotated content. In practice this bites hardest
on exactly the workloads KubeRay exists to serve: multi-day training runs, long-lived Ray
Serve deployments, and batch inference clusters that stay up for weeks. Those are the
clusters that generate enough log volume to rotate in the first place, and they are also
the ones where a postmortem needs the *early* logs — the ones that rotated away first —
to explain a failure that surfaced much later. A collector that only reliably captures the
final window of logs is most reliable precisely when it is least needed.

---

## Issue Selection Checklist

Evaluated honestly, including the checks that are not a clean pass.

| # | CodePath check | Verdict | Evidence / caveat |
|---|---|---|---|
| 1 | I understand the problem | Pass | The issue body states the mechanism ("log collector only reads and uploads all the files under ray logging directory honestly at process termination, which means rotated log content is not recoverable") and links the Ray rotation docs. My restatement is the Problem Summary above, and the Preliminary Codebase Orientation section traces the mechanism to specific functions. |
| 2 | Scope fits 3–4 weeks | Pass, with managed risk | This is labelled P2/enhancement, not `good first issue`, and the issue author said outright that it is not straightforward, so I am not pretending it is small. What makes it bounded is the shape of the change: it is **additive**, a new periodic path running alongside the existing shutdown and `prev-logs` paths rather than a rewrite of either, and the hard sub-problem (not re-uploading what was already uploaded) already has a working precedent in the repository that I expect to reuse rather than invent. The risk I am actively managing is the segment-identity question in the Risks section; if that turns out to need a mechanism nobody has built, I will renegotiate scope on the issue rather than quietly overrun. |
| 3 | Matches my skills or skills I can learn quickly | Pass | Match: backend file-lifecycle reasoning, idempotency, restart safety, and test design, all of which carry over from Python backend work. Learn quickly: Go (goroutines, tickers, `filepath.WalkDir`, table-driven tests), Kubernetes sidecars, and shared volume semantics. The correctness reasoning and the language are not both new at once, which is what makes the stretch realistic. |
| 4 | Issue is active and claimable | Pass, with a caveat | [#4830](https://github.com/ray-project/kuberay/issues/4830) is open, filed by `dentiny` (the issue author) on 2026-05-12, labelled `P2`/`enhancement`. The GitHub sidebar shows **Assignees: No one assigned** and **Development: No branches or pull requests**. The repository itself is active: 31 commits on `upstream/master` in the 30 days before my last fetch, 35 distinct authors in 90 days, 19 commits touching `historyserver/`. The caveat is [PR #4983](https://github.com/ray-project/kuberay/pull/4983), which touches adjacent code without claiming this issue. I have not claimed assignment and have not asked to be assigned. |
| 5 | Helpful context exists | Pass | The issue author wrote the Ray log-rotation documentation and linked it from the issue, so the upstream behaviour is documented rather than guessed. There is prior discussion in [#4825](https://github.com/ray-project/kuberay/issues/4825) on History Server usage patterns, an earlier comment endorsing the direction, and the collector code carries substantial explanatory comments (for example the worked example above `isFileAlreadyPersisted`). |
| 6 | Clear setup documentation exists | Pass | [`CONTRIBUTING.md`](https://github.com/ray-project/kuberay/blob/master/CONTRIBUTING.md) documents PR conventions (`subject: message`, squash-and-commit, and that tests/docs are the feature owner's responsibility) and points to per-subproject `DEVELOPMENT.md`. `historyserver/DEVELOPMENT.md` gives a full kind + MinIO + History Server walkthrough, including how to generate a dead session and validate both the dead and live paths. |

---

## Repository Activity and Claimability Evidence

Verified on 2026-07-26 against my local clone of `ray-project/kuberay`, whose `upstream`
remote was last fetched on 2026-07-22 at `6fe0223c`. Because the clone is a few days
stale, the commit windows below are relative to that fetch, not to today.

| Signal | Value |
|---|---|
| `upstream/master` HEAD | `6fe0223c` — "Rename TargetClusterChanged for clarity (#5022)" |
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

1. **Earlier project-member signal that the direction is wanted** —
   https://github.com/ray-project/kuberay/issues/4830#issuecomment-4523841501
   A project member (`JiangJiaWei1103`) engaged with the request before I was involved.
   This predates my involvement and is what convinced me the feature was worth picking up
   rather than a request that had already been declined.

2. **My introduction on the issue** —
   https://github.com/ray-project/kuberay/issues/4830#issuecomment-5052716814
   I said I wanted to work on the issue and described how I planned to start.

3. **The issue author asking me to share a design first** —
   https://github.com/ray-project/kuberay/issues/4830#issuecomment-5052727165
   `dentiny`, who filed the issue, made the point that this is not a straightforward task
   and that a design should be discussed before implementation. I took that seriously
   rather than treating it as optional.

4. **My detailed design comment** —
   https://github.com/ray-project/kuberay/issues/4830#issuecomment-5052938252
   This is the design summarised in the "Proposed Direction" section below. I posted it
   publicly on the issue with the author and project members tagged, so the discussion
   happens in the open where anyone reviewing can see it.

**Current state of the conversation:** feedback on the full design is pending. To be
explicit about what that does and does not mean:

- The issue is **not** assigned to me, and I have not asked for it to be.
- Nobody has approved my proposed design. Pending feedback is exactly that — pending —
  and I am not treating the absence of a reply as agreement.
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

- `WatchPrevLogsLoops()` — runs on **both** head and worker nodes. It scans
  `/tmp/ray/prev-logs` on startup and then uses three `fsnotify` watchers to pick up new
  session/node directories, so leftover logs from a previous run get uploaded.
- Head-only goroutines: `WatchSessionLatestLoops()`, `FetchAndStoreClusterMetadata()`,
  `FetchAndStoreTimezone()`, and `PollAdditionalEndpointsPeriodically()`.
- On stop, `Run` calls `processSessionLatestLogs()` (both roles), then
  `processAdditionalEndpoints()` (head only), then closes `ShutdownChan`.

### Preliminary finding 1 — the live-session upload really is shutdown-shaped

`processSessionLatestLogs` (`collector.go:91`) resolves the `session_latest` symlink,
reads the node ID from `/tmp/ray/raylet_node_id`, and `filepath.WalkDir`s the session
`logs/` directory, uploading every regular file it finds via `processSessionLatestLogFile`
(`collector.go:176`), which does a whole-file `os.ReadFile` followed by
`Writer.WriteFile`. As far as I can tell from reading `Run`, this function is called from
exactly one place: the shutdown path. There is no ticker, no watcher, and no other caller
that uploads live-session log content while the cluster is running. This matches the
reporter's description of the mechanism.

### Preliminary finding 2 — there is already a dedupe pattern worth reusing

The `prev-logs` path solves the duplicate-upload problem in a way that looks directly
reusable. `processPrevLogFile` (`collector.go:615`) uploads a file and then `os.Rename`s
it into `/tmp/ray/persist-complete-logs/{sessionID}/{nodeID}/logs/...`, and
`isFileAlreadyPersisted` (`collector.go:517`) checks for the file's presence in that
directory before uploading. The marker lives on the shared volume, so it survives a
collector restart. Two details matter for #4830: the marker path is derived from the
*filename*, which is exactly the thing that changes when a backup is renamed from `.1` to
`.2`; and `processPrevLogsDir` deletes the whole node directory when it finishes
(`collector.go:607`), which is fine for a dead session but would be wrong for a live one.

### Preliminary finding 3 — there is an existing periodic-loop pattern to copy

`PollAdditionalEndpointsPeriodically` (`poll.go:26`) is the shape a rotated-log scanner
should follow: do one pass immediately, then `time.NewTicker(interval)` in a
`for { select { case <-r.ShutdownChan: ...; case <-ticker.C: ... } }` loop. If I add a
periodic scanner, following this structure keeps it consistent with code that already
exists rather than inventing a new lifecycle.

### Preliminary finding 4 — the periodic-push plumbing exists but is unused

`RayLogHandler` declares `PushInterval`, `LogBatching`, `LogFiles`, `logFilePaths`, and
`LogDir` (`collector.go:23–47`). `PushInterval` is fed by the `--push-interval` flag in
`historyserver/cmd/collector/main.go` (default one minute) and plumbed through
`runtime.NewCollector` (`runtime.go:18`), but grepping the non-test sources shows it is
only ever *assigned*, never read by the log-collector loops. `r.LogDir` appears in `Run`
only in a commented-out line (`collector.go:50`). This is preliminary and I want to
confirm it upstream, but if it holds, there is already a configured interval
looking for a consumer, and I should ask whether reusing `--push-interval` is preferred
over adding a new flag.

### Preliminary finding 5 — what Ray actually does on rotation

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
  so `.1` → `.2` does not produce a second copy of the same bytes;
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
| `historyserver/pkg/collector/logcollector/runtime/logcollector/collector_test.go` | 241 | Existing test harness — `MockStorageWriter` (an in-memory `storage.StorageWriter`) and `setupRayTestEnvironment`, which builds `prev-logs` / `persist-complete-logs` trees under a temp root. New rotation tests should extend this rather than build a parallel harness. |
| `historyserver/pkg/collector/logcollector/runtime/runtime.go` | 60 | `NewCollector` wires config into `RayLogHandler`; any new interval or toggle is threaded through here. |
| `historyserver/pkg/collector/types/types.go` | — | `RayCollectorConfig`, where a new configuration field would be declared. |
| `historyserver/cmd/collector/main.go` | 192 | Collector entrypoint and flag parsing; already defines `--push-interval` and reads `RAY_COLLECTOR_*` environment variables, so it is the natural place for scan configuration. |
| `historyserver/pkg/utils/constant.go` | 38 | `GetTmpRayRoot`, `GetRayPrevLogsPath`, `GetRayPersistCompletePath`, `GetRaySessionLatestPath`, `GetRayNodeIDPath` — every path the collector touches resolves through here, including the `RAY_TMP_ROOT` override the tests rely on. |
| `historyserver/pkg/utils/utils.go` | 174 | `GetLogDirByNameID` builds the object key layout `{root}/{cluster}_{ns}/{session}/logs/{nodeID}/...`; rotated segments must land in the same layout so the History Server UI can read them. |
| `historyserver/pkg/storage/interface.go` | 26 | `StorageWriter` is only `CreateDirectory` + `WriteFile`. Worth confirming upstream whether whole-file rewrites are acceptable for large rotated segments or whether an append/streaming concern needs raising. |
| `historyserver/DEVELOPMENT.md` | — | The kind + MinIO + History Server walkthrough I will follow in Phase II to actually observe rotation. May need a note if new configuration is added. |
| `historyserver/config/`, `historyserver/test/e2e/` | — | Collector sidecar configuration and end-to-end tests; likely touched only if a new flag or environment variable is introduced. |

---

## Related Issue / PR Context

- **[PR #4983](https://github.com/ray-project/kuberay/pull/4983)** — modifies node ID
  handling and leftover-log handling in the same collector area. This overlaps with the
  code I would be changing: node ID resolution feeds the storage key built by
  `GetLogDirByNameID`, and "leftover logs" is the `prev-logs` path whose dedupe mechanism
  I want to reuse. Whatever I implement has to be rebased on top of it, and if it changes
  how the persisted-marker directory works, my design changes with it. I am treating this
  PR as a dependency to track, not a blocker.
- **[Issue #4825](https://github.com/ray-project/kuberay/issues/4825)** — "Question:
  what's the usage pattern for history server", referenced from #4830 by the same
  reporter. Useful background on how the History Server is expected to be used, which
  matters for deciding whether a lossy-but-cheap scanner is acceptable.
- **[Ray log rotation documentation](https://docs.ray.io/en/latest/ray-observability/user-guides/configure-logging.html#log-rotation)**
  — the authoritative description of the rotation behaviour, written by the issue reporter
  and linked from the issue body.

---

## Preliminary Definition of Done / Acceptance Criteria

These are the outcomes I proposed. They are **not** agreed criteria — every one of them is
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

Criteria 2–6 are the ones I would defend hardest, because they are about not breaking or
double-writing anything. Criterion 1 is the one most likely to need rewording once there
is a shared view of how much loss is acceptable.

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
  not have a settled answer yet — this is the single most important thing to resolve in
  Phase II.
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
- [x] All six CodePath issue-selection checks evaluated honestly, including the scope check
      where the risk is real and managed
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
- [ ] Feedback on the design — pending; will be incorporated if received
- [ ] Issue assignment — not requested, not granted

---

## Next Steps

### Completed in Phase I

Fork and branch setup, issue-selection evidence, community introduction, the read-only
code investigation summarised above, and the public design comment. This document is the
Phase I deliverable.

### Phase II (starting now)

Phase II is investigation and reproduction, and it proceeds now. None of it depends on
feedback arriving first: reproducing the loss and characterising the rotation behaviour is
useful evidence whichever mechanism is eventually chosen, and it is what will let me argue
for a mechanism with data instead of assertion.

1. Stand up the local environment from `historyserver/DEVELOPMENT.md` — kind cluster,
   KubeRay operator, MinIO, History Server — and confirm the normal log path works end to
   end before touching rotation.
2. Force rotation with small values (`RAY_ROTATION_MAX_BYTES` and
   `RAY_ROTATION_BACKUP_COUNT` set low) so segments rotate in seconds instead of at
   512 MB, and record the exact sequence of renames and deletions that Ray performs.
3. Demonstrate the actual loss: show that a segment created and deleted while the cluster
   is running never appears in MinIO, with concrete evidence rather than inference from
   code reading.
4. Confirm or correct preliminary findings 1 and 4 — specifically, that
   `processSessionLatestLogs` has no non-shutdown caller, and that `PushInterval` is
   genuinely unused by the log-collector loops.
5. Investigate segment identity across renames (inode, size, hash) and pick a candidate
   with evidence behind it.
6. Read PR #4983 in detail and work out whether to wait for it or build on it.
7. Verify empirically which log components actually rotate in the KubeRay test image, and
   under what filenames, rather than relying on the documentation's general statement.

### Phase III (after Phase II validation)

Implementation begins once Phase II has validated the behaviour and resolved the key
design risks — specifically, that the loss reproduces as described, and that segment
identity across renames has a workable answer. Waiting for that is a sequencing decision
based on evidence, not a hold on anyone else's reply.

Maintainer feedback remains pending and will be incorporated into the design if it
arrives; if it changes the mechanism, I will update this document and say so plainly
rather than quietly reframing what I already wrote. If nothing arrives by the time Phase
II is complete, I will follow up on the issue with the reproduction results attached,
which is a better prompt for a reply than a second ping — and I will not treat a lack of
response as agreement with my design.

---

## Resources Used

- KubeRay issue #4830 — https://github.com/ray-project/kuberay/issues/4830
- KubeRay issue #4825 — https://github.com/ray-project/kuberay/issues/4825
- KubeRay PR #4983 — https://github.com/ray-project/kuberay/pull/4983
- KubeRay repository — https://github.com/ray-project/kuberay
- KubeRay `CONTRIBUTING.md` — https://github.com/ray-project/kuberay/blob/master/CONTRIBUTING.md
- KubeRay History Server development guide — https://github.com/ray-project/kuberay/blob/master/historyserver/DEVELOPMENT.md
- Ray log rotation and logging configuration — https://docs.ray.io/en/latest/ray-observability/user-guides/configure-logging.html#log-rotation
- Ray log persistence on Kubernetes — https://docs.ray.io/en/latest/cluster/kubernetes/user-guides/persist-kuberay-custom-resource-logs.html
- Go `fsnotify` — https://pkg.go.dev/github.com/fsnotify/fsnotify
- Go `path/filepath` (`WalkDir`, `EvalSymlinks`, `Rel`) — https://pkg.go.dev/path/filepath
- Kubernetes sidecar containers and shared volumes — https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/
