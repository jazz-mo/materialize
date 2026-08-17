# Design: Replica resource usage observations

## Summary

Replica resource usage is visible only as periodic samples taken by the orchestrator, roughly one
per minute, landing in `mz_cluster_replica_metrics_history`. A spike between two samples is
invisible, so the usage of a hydration episode that starts and finishes inside one sampling gap
cannot be recovered at all.

This adds a replica-local, higher-cadence surface: `mz_introspection.mz_cluster_resource_usage`
reports what each measurement source says about each replica process, one row per
`(process_id, source, metric)`.

## Mechanism, not policy

The relation surfaces uninterpreted observations. It does not decide which number is "the" memory
usage of a replica, and it never combines two sources into a third figure. That separation is the
central design decision, and it is a reaction to what fusing costs.

Sources measure overlapping but distinct quantities. `cgroup memory.current` is the accounting
that limit enforcement and the OOM killer act on. `/proc/self/status`'s `VmRSS` is the process's
resident set. `getrusage`'s `ru_maxrss` is a high-water mark of the resident set, so it excludes
swap. These disagree, and by more than rounding.

Measured on a staging replica: `VmRSS` runs a roughly constant **96 MiB above** the replica's own
cgroup charge, holding within a 7% spread across replicas whose `memory.current` spans 36 MiB to
400 MiB, and holding at the same constant between `ru_maxrss` and `memory.peak`. A page is charged
to whichever cgroup first faulted it in, so this binary's own resident text, faulted in by the
runtime that unpacked the image, counts in `VmRSS` while being charged elsewhere. `memory.stat`
reports `file` as 0 throughout, confirming no page cache is charged to the replica at all.

The offset is additive, so its relative weight is worst exactly where sizing is tightest: taking
`ru_maxrss` as the replica's memory footprint overstates the enforced figure by 2.75x on an idle
replica and 1.20x on a loaded one. `cgroup memory.current` matches what cAdvisor reports for the
same container to within 1.5%, so it is the figure a limit acts on, and `VmRSS` minus `rss_file`
is the figure the process is responsible for. Neither is "the" memory usage. A single fused number
has to pick one and hide the discrepancy, which is why this relation reports both and why
`rss_anon`, `rss_file` and `rss_shmem` are surfaced alongside: they make the gap attributable
rather than mysterious.

Fusing also destroys information irreversibly, and does it at the layer least able to judge. A
maximum over `ru_maxrss` and `VmRSS + VmSwap`, taken to preserve an ordering that a pair of column
names implied, yields a number that neither source reported. Combining sources belongs in views,
where it can be changed without a migration and where a reader can see what was combined.

The word "memory" already means at least three different things across this codebase, and
`mz_cluster_replica_metrics.disk_bytes` already reports disk plus swap because a consumer wanted
it displayed that way. Both are the result of interpreting at the measurement layer.

## Long rows

Columns are `(process_id, source, metric, value)` rather than one column per metric.

A wide row retracts and re-inserts on any field's change, so its churn is set by its noisiest
field: a filesystem-usage number that moves every sample would drag a memory peak that has not
moved in an hour through a retraction every time. Long rows churn only what changed.

Long rows also make the absence of an observation representable without a sentinel. A metric the
replica cannot read is simply not present, which is distinct from a metric that read zero, and
which needs no in-band `NULL` or magic value. Adding a metric as kernels gain interface files is
then a change to the sampler alone, with no catalog migration.

## Peaks

Both instantaneous values and high-water marks matter, and a peak is just another observation.

A peak cannot be derived downstream from this relation, and that is a property of the shape rather
than an oversight. The collection holds the current value per key, because each sample retracts
the previous one; a `reduce` computing a maximum over it sees one value per group and returns it.
Recovering a maximum over time would require retaining the samples, and a compute log collection
is an arrangement with no retention, window, or TTL mechanism available to it, so a retained
sample history would grow without bound for the process's lifetime.

So peaks are reported, not derived. Most of them cost nothing to report, because the kernel
already maintains them:

* `cgroup memory.peak` and `memory.swap.peak` are exact high-water marks of the cgroup accounting,
  maintained per cgroup for the container's lifetime. They are unaffected by how rarely we read
  them, by the sampler being disabled and restarted, and by `environmentd` restarting.
* `rusage max_rss` is already a high-water mark, so there is nothing to maintain. Note it is a
  lagging one: the kernel refreshes it at internal checkpoints, not on every fault, and it has been
  observed reading below the concurrent `VmRSS` in the same sample. Monotone, but not exact.

Only a source with no kernel-side peak gets one folded in this process, currently the scratch
filesystem's usage, and swap below the kernel version that provides `memory.swap.peak`. Note that
current nodes integrate disk as swap and have no scratch filesystem, so `statvfs` contributes no
observations there and disk usage arrives as `swap_current` instead. That is also why
`mz_cluster_replica_metrics.disk_bytes` reports disk plus swap: it is a filesystem-named column
carrying a swap figure. Those are
maxima over samples and therefore lower bounds, and they are reported under a distinct `_peak`
metric name so a reader can tell an exact peak from a sampled one. They are folded in the sampler
rather than in the logging operator so that they survive a teardown and rebuild of the logging
dataflow.

Peaks are never reset. A peak that resets is not composable: whoever reads it first consumes it, a
retried read loses it, and two consumers reading at different times disagree about the same
episode. The kernel's `.peak` files are resettable by writing to them, and we deliberately do not,
for the same reason.

Peaks die with the process. `mz_cluster_replica_metrics_history` is the surface that outlives a
replica, being persist-backed and retained for 30 days; this relation is replica-local.

### Measured: what the peaks recover

A replica hydrating a 60M-row index on a node whose RAM limit is 3.79 GiB, sampled once hydration
had finished and the arrangement had settled:

| | instantaneous | peak | understatement |
|---|---|---|---|
| `heap` | 606.1 MiB | 5.425 GiB | 9.2x |
| `memory_current` | 435.9 MiB | 3.790 GiB | 8.9x |
| `swap_current` | 150.9 MiB | 2.726 GiB | 18.5x |

`memory_peak` lands within two pages of `memory_max`, the signature of a cgroup pinned at its
ceiling with the remainder spilling to swap. Yet the instantaneous reading puts the replica at
10.7% of its RAM limit. Both numbers are correct; only one of them answers "was this replica
memory-constrained". A periodic sample taken after the episode sees the 10.7% and nothing else,
which is the gap this relation exists to close.

The reporting path is slowest exactly here. While the arrangement was building, a query against
this relation took 173 seconds, then 34 seconds, then 34 milliseconds once the worker went idle:
the logging operator runs only when its worker schedules it, and a saturated worker barely does.
The peaks still arrive intact, because the sampler folds them on the metrics task rather than in
the operator. The report is late; no observation is lost.

The peaks are per-dimension and must not be added. Sampling the same replica minutes later showed
`memory_current` up 2.46 GiB while `swap_current` fell 616 MiB, both peaks unchanged: the
dimensions trade against each other as pages swap in and out, so the two maxima need not have
occurred at the same instant. `memory_peak + swap_peak` bounds the simultaneous total from above
rather than measuring it, and cgroup v2 exposes no combined memory-plus-swap peak that would.

### Bracketing the limiter's quantity

The memory limiter enforces `vm_rss + vm_swap` against `--heap-limit`, which is the replica's
memory limit plus its disk limit. Limiting the sum rather than the dimensions separately is
deliberate: swap is not eagerly paged back in, so a replica can sit pinned at its RAM ceiling
indefinitely while its total footprint keeps growing, and Kubernetes offers no way to express a
swap limit anyway. Being at `memory.max` is therefore normal operation under disk-as-swap, not a
danger signal.

That leaves "how close did this replica come to a limiter kill" answerable only from below.
`heap_peak` is folded from samples of the sum, so it can miss a spike shorter than the sampling
interval, and it is a lower bound on the true peak.

There is no matching upper bound, and the two candidates both fail for reasons worth recording,
since both look correct until measured.

`memory_peak + swap_peak` is cgroup-scoped. It bounds the peak of the *cgroup's* memory plus swap,
which is a different quantity from the limiter's, because the cgroup is not charged for the
replica's resident file-backed pages. Two effects pull it in opposite directions: excluding
`rss_file` makes it smaller, while summing two maxima that need not have co-occurred makes it
larger. Which effect wins depends on the workload, so the ordering is not merely wrong but
unstable. Measured on two staging replicas:

| replica | `heap_peak` | `memory_peak + swap_peak` |
|---|---|---|
| idle, no swap | 525.30 MiB | 426.65 MiB |
| after swapping 2.7 GiB | 5.425 GiB | 6.516 GiB |

The idle case inverts, the loaded case does not. Peaks of two different quantities do not bracket
either of them, and a relationship that reverses under load is worse than one that is consistently
wrong, because it survives casual checking.

There is a third reason, independent of the other two: `memory_current` and `swap_current` double
count. A page swapped out and later read back stays in the swap cache, resident in memory with its
swap slot still allocated so it can be evicted again without rewriting. It is charged as `anon` in
`memory_current` and simultaneously charged in `swap_current`. Measured on a staging replica whose
node carried essentially no other swap: `swap_current` 150.94 MiB against `vm_swap` 87.51 MiB, a
gap of 63.43 MiB, matching the node's `SwapCached` of 59.52 MiB to within the ~4 MiB of swap
belonging to other processes there. The `swapcached` metric reports this directly.

The limiter's `vm_rss + vm_swap` does not have this problem, and not by accident. `VmSwap` counts
swap entries in the process page tables, so it drops as soon as a page is read back, while the
page then appears in `VmRSS`. Exactly one of the two terms counts a swap-cached page. Switching
the limiter to cgroup counters would introduce the double count and make it kill replicas early.

`rusage max_rss` is process-scoped and so has the right shape, but it lags. In one consistent
sample the same replica reported `max_rss` 521.14 MiB against `vm_rss` 525.30 MiB, the high-water
mark reading 4.16 MiB *below* the current value. The kernel refreshes `ru_maxrss` at internal
checkpoints rather than on every fault, so it is monotone but not exact, and it can understate both
the true peak and the present reading. It is a lower bound too.

So the relation reports `heap_peak` as a lower bound and stops there. Combining it with a
cgroup-scoped peak to manufacture an upper bound is exactly the cross-source fusing this design
exists to prevent, and it produces a contradiction on the first real measurement.

`memory.events` reported `max` as 0 throughout, despite the cgroup demonstrably reaching its limit,
because reclaim succeeded by swapping rather than failing. `events_oom_kill` was likewise 0, which
is the disk-as-swap arrangement working as intended. So `events_max` is not a reliable "hit the
limit" signal where swap is configured; `memory_peak` reaching `memory_max` is. Both are reported
without interpretation, and choosing between them is a query's job.

## Where the sampling happens

In `mz_metrics::usage`, on the periodic task that already samples `rusage` and lgalloc stats. That
task runs on the tokio runtime, independent of the timely workers.

Folding a maximum inside the compute logging operator would sample exactly where sampling is least
reliable: a logging operator runs only when its worker schedules it, and a worker saturated by
hydration is precisely the case whose peak we want.

The logging operator downgrades its capability on its own timer rather than on the sampler's. Were
the sampler to drive progress, disabling it, which `mz_metrics_usage_refresh_interval = 0` is meant
to allow, would freeze the collection's frontier and wedge every query over the relation along with
the `mz_catalog_server` indexes built on it.

## Cost

The sampler reads `getrusage`, `/proc/self/status`, `statvfs`, and a handful of cgroup interface
files per tick, at `mz_metrics_usage_refresh_interval` (5s by default), kept separate from
`memory_limiter_interval`, which governs OOM-kill behavior and must not be retuned for
introspection's sake.

None of these is proportional to heap size. `VmRSS` and `VmSwap` come from per-mm counters and
cgroup usage and peak figures are `page_counter` reads, so all are O(1). Measured on a 32-core
Linux host, reading `/proc/self/status` costs 4.67 µs with an empty heap and 4.74 µs with 32 GiB
resident, and `getrusage` and `statvfs` are 0.22 µs and 0.41 µs at both sizes. This is worth
stating explicitly because the neighbouring `/proc` files behave very differently: `smaps_rollup`
and `numa_maps` walk page tables, and on the same host cost 137 ms and 142 ms at 32 GiB, rising
superlinearly. Those files are not read here, and should not be added to this path.

## Alternatives considered

* **Extend `/api/usage-metrics` and `mz_cluster_replica_metrics_history` instead.** That surface
  already has the shape argued for here: append-only observations with an `occurred_at`, a
  latest-value view over them, 30-day retention with TTL truncation, and survival across replica
  restarts. A maximum over a window is a plain `max(...) GROUP BY` there. It is the right home for
  coarse, long-retention, cross-replica history, and it should gain the same uninterpreted cgroup
  fields. It is the wrong home for high-cadence sampling: the transport is one HTTP round trip per
  process per interval driven from `environmentd`, which is why its poll interval is 60s. The two
  surfaces are complementary rather than alternatives.
* **Reuse `mz_cluster_prometheus_metrics`.** The gauges appear there automatically, so this needs
  no catalog change at all. Rejected as the primary surface: values are untyped `double`, the
  relation is a debugging escape hatch rather than a documented contract, and it is gated by its
  own scrape-interval dyncfg.
* **Retain samples and compute maxima in SQL.** Rejected because a compute log collection has no
  retention mechanism, as described under Peaks.
* **Push samples from the sampler into the logging dataflow through a queue.** This would enable
  maxima over an arbitrary window rather than only since process start. Rejected for now: it needs
  cross-thread plumbing that no logging producer currently uses, it makes the collection's frontier
  depend on a tokio task, and it does not solve the retention problem that makes windowed maxima
  expensive. Kernel-maintained peaks cover the since-start case at no cost, and the coarse windowed
  case is already served by `mz_cluster_replica_metrics_history`.
