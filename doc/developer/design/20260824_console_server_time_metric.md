# Console SQL Shell: report server time alongside round-trip time

- Associated: [CS-218](https://linear.app/materializeinc/issue/CS-218/console-response-time-metric-should-show-actual-database-time)

## The Problem

The Console SQL Shell footer reports a single number, `Returned in 148ms`. During
demos this undersells Materialize: the number is dominated by network round trip
and browser work, so a query that Materialize answered in about a millisecond
presents as a hundred and change.

The metric is not wrong. `CommandResult.tsx` computes it via
`calculateCommandDuration` (`console/src/platform/shell/timings.ts`), which
subtracts two client-side `performance.now()` stamps: `commandSentTimeMs`, taken
on the shell FSM's `SEND` transition, and `endTimeMs`, taken in
`completeCommandResult` when the FSM processes the completion frame. It therefore
spans WebSocket send, network out, server work, network back, JSON parse, and FSM
dispatch, plus any main-thread stall in between. The existing tooltip says so
explicitly:

> "The total time to submit, execute, receive, and render the query results over
> the network."

So this is not a bug report. It is a request to surface a number the client
cannot compute, because **the server never tells the client how long it spent**.

## Success Criteria

- A user running a query in the Shell can see how much of the elapsed time
  Materialize was responsible for, distinct from network and browser time.
- The Shell does not overstate Materialize's speed. Whatever is shown must remain
  true when the server is slow, not only when it is fast.
- No behaviour change for any existing WebSocket API client that does not ask for
  the new information.
- The Shell continues to work, and to look deliberate, against environments that
  predate this change.

## Out of Scope

- **Streaming commands.** A `SUBSCRIBE` has no completion, and the Shell already
  shows no duration for one until it ends. That stays true.
- **A latency breakdown.** Splitting server time into queueing, optimization and
  execution is deferred; see Follow-up work.
- **The HTTP SQL API.** Parity is desirable but is not required to close CS-218,
  and bundling it widens the blast radius of a public-API change.
- **Query Insights thresholds.** See Open questions.

## Solution Proposal

The server measures the interval from receiving a statement to having its
response ready, and reports it to clients that opted in. The Shell renders both
numbers:

```
1.2ms server · 148ms total
```

### What "server time" means, and why not "execution time"

The statement lifecycle already defines five points
(`src/adapter/src/statement_logging.rs`): `ExecutionBegan`,
`OptimizationFinished`, `StorageDependenciesFinished`,
`ComputeDependenciesFinished`, `ExecutionFinished`. Separately,
`LifecycleTimestamps.received` (`src/adapter/src/session.rs`) marks arrival.

The narrowest interval, `ExecutionBegan` to `ExecutionFinished`, produces the
smallest and most flattering number. It is the wrong choice. The gap between
*received* and *execution-began* is time queued behind the coordinator, which is
a single serialized task per environment. In several 2026 incidents that gap was
the dominant term: every query got slower while no individual query was slow. A
metric that excluded it would display roughly a millisecond while a user waited
seconds, making the Console actively misleading in precisely the situation where
someone is looking at it to understand a problem.

So the reported interval is **received to response-ready**, and it is labelled
*server time* rather than *execution time*. The demo value survives: on a healthy
system an indexed peek is still low single-digit milliseconds against a
three-digit round trip, which is the point we want to make.

### Why the statement-logging machinery cannot supply this

The natural implementation is to reuse the lifecycle timestamps. It does not
work. `record_statement_lifecycle_event`
(`src/adapter/src/coord/statement_logging.rs`) requires a `StatementLoggingId`,
which exists only for statements sampled into the statement log, and it is
additionally gated on the `ENABLE_STATEMENT_LIFECYCLE_LOGGING` dyncfg. It is
doubly conditional, and one production environment samples on the order of 0.06%
of statements. A number that appears in the Shell on every query needs its own,
unconditional path.

### Clock

The measurement uses a monotonic `std::time::Instant` taken at receipt and at
response-ready, reported as microseconds.

`EpochMillis` (`src/ore/src/now.rs`), which the adapter's `NowFn` returns, is
unsuitable on two counts. It has millisecond granularity, so a fast peek renders
as `1ms` or `0ms`, and `0ms server` reads as broken rather than fast. And it is
wall-clock, so an NTP step can produce a negative duration. `Instant` is
monotonic by construction and sub-millisecond, matching the client half, which
already uses `performance.now()`.

### Protocol

Add a `WebSocketResponse` variant carrying the measurement, emitted immediately
before `CommandComplete` for each statement.

`WebSocketResponse` is adjacently tagged (`#[serde(tag = "type", content =
"payload")]`), so a new variant is additive on the wire and cannot alter the
encoding of existing messages. Widening `CommandComplete`'s payload from its
current bare string to an object was rejected for the opposite reason: it would
break every existing client.

The new message is **opt-in**. The WebSocket handshake already accepts an
`options` map of session variables (`src/environmentd/src/http/sql.rs`,
documented in `doc/user/content/integrations/websocket-api.md`). A client that
does not set the option receives a byte-identical stream to today. This matters
because the WebSocket SQL API is a documented public interface whose message-type
table is presented as exhaustive, with no clause telling clients to tolerate
unknown types. Rather than assume third-party clients are permissive, we require
them to ask.

The Console opts in at connection time. It is safe against a server that ignores
the option: its dispatch is a `switch` with no `default` arm wrapped in a
`try/catch`, so unknown message types are already discarded silently.

### Errors and streaming

Failed statements report server time. The Shell already stamps completion on the
error path (`addErrorDuringCommandInProgress` calls `completeCommandResult`) and
renders the duration in red. A statement that fails after three seconds is
exactly when the split between server and network matters.

Streaming commands report nothing, matching today's behaviour.

### Display

```
1.2ms server · 148ms total          ⓘ
```

Against an environment that does not send the message, the Shell renders today's
line unchanged:

```
Returned in 148ms                   ⓘ
```

The fallback is not an error state and must not look like one. The tooltip is
rewritten for both cases: today's wording describes the total accurately and
would become misleading next to a second number.

### Implementation touch points

Three CODEOWNERS scopes in one repository, so this stacks rather than requiring
cross-team coordination:

1. `@MaterializeInc/adapter` — the session option, the `Instant` pair on the
   response path, and the new `WebSocketResponse` variant.
2. `@MaterializeInc/console` — opt in at connect, handle the message, prefer the
   server value with fallback, render both, rewrite the tooltip. Adds
   `timings.test.ts`; there is no test for that module today.
3. `@MaterializeInc/docs` — the new message type in the WebSocket API reference,
   and a section in `doc/user/content/console/sql-shell.md`, which is 27 lines
   and does not currently mention the metric at all.

Step 1 must land first. Steps 2 and 3 are independent of each other.

## Minimal Viable Prototype

A branch that hardcodes a plausible server time in the Console and renders the
two-number footer, shared as a screenshot, is enough to settle the display
question before any protocol work begins. The wording and ordering are the parts
most likely to attract revision, and they are the cheapest to change early.

## Alternatives

**Replace the total with server time.** Best demo optic and closest to the
literal ticket title. Rejected: for a browser talking to a remote environment,
round-trip time is a real component of what the user experiences, and hiding it
trades one form of inaccuracy for another.

**Tighten the client-side measurement instead.** Move the stamps closer to the
socket so JS work falls outside them. Cheap, one CODEOWNERS scope, no protocol
change. Rejected as insufficient: it reduces noise but still measures a
round trip, so it does not answer the question CS-218 asks.

**Query `mz_recent_activity_log` after execution.** No protocol change. Rejected:
statement logging is sampled and throttled, requires `mz_monitor`, and adds a
round trip to display a latency number.

**Carry the timing in a `Notice`.** No protocol change at all, and clients
already tolerate arbitrary notices. Rejected: notices are user-facing
informational messages, the Shell renders them into output, and every other
client displaying notices would inherit timing noise.

**Emit the new message unconditionally.** Simpler than an opt-in, and safe for
any client that ignores unknown types. Rejected on public-API grounds: the
documented type table reads as exhaustive, and we would be changing observable
output for every consumer to serve one.

## Open questions

**Should Query Insights follow the new number?**
`PlanInsightsNotice.tsx` uses the same `calculateCommandDuration` helper to
decide when to surface Query Insights. If the displayed metric changes basis, the
insights threshold silently moves with it. The recommendation is to pin Query
Insights to round-trip time: a user who waited three seconds cares that they
waited three seconds, wherever the time went. This should be an explicit choice
rather than an inherited side effect.

**Should the public WebSocket API gain a forward-compatibility clause?**
`doc/user/content/integrations/websocket-api.md` enumerates message types with no
statement that clients should ignore unrecognised ones. That absence is what
forces this design to be opt-in. Adding such a clause would not help existing
deployed clients, but it would let future additions be unconditional. Worth doing
independently of this work.
