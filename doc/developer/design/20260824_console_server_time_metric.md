# Report server time for SQL statements

- Associated: [CS-218](https://linear.app/materializeinc/issue/CS-218/console-response-time-metric-should-show-actual-database-time)

## The Problem

The Console SQL Shell footer reports a single number, `Returned in 148ms`. It is
dominated by network round trip and browser work, so a query Materialize answered
in about a millisecond presents as a hundred and change. The original report
(CS-218, from a demo context) is that this undersells the product.

The metric is not wrong. `calculateCommandDuration`
(`console/src/platform/shell/timings.ts:28-50`) subtracts two client-side
`performance.now()` stamps. For the first statement of a command the start is
`commandSentTimeMs`, taken on the shell FSM's `SEND` transition
(`console/src/platform/shell/machines/webSocketFsm.ts:224`). For every subsequent
statement the start is the *previous* statement's `endTimeMs`
(`timings.ts:41-43`), so the reported figure is an inter-completion delta. The end
is `endTimeMs`, taken in `completeCommandResult` (`webSocketFsm.ts:97-99`) when the
FSM processes the completion frame. The existing tooltip says as much:

> "The total time to submit, execute, receive, and render the query results over
> the network."

So this is not a bug. It is a request for a number no client can compute, because
**Materialize never tells any client how long it spent**. That gap is not specific
to the Console: a coding agent over MCP, a `psql` user, and a dbt run are all
equally unable to separate their own latency from ours.

## Success Criteria

- A client can determine how much of a statement's elapsed time Materialize was
  responsible for, separately from network and client time.
- The number does not overstate Materialize's speed. Whatever is reported stays
  true when the server is slow, not only when it is fast.
- The mechanism is available to non-Console clients, including agents, without
  further protocol work.
- No behaviour change for clients that do not opt in.
- The Console continues to work, and to look deliberate, against environments that
  predate this change.

## Out of Scope

- **Writes.** See "Where the boundary sits"; the interval cannot be defined
  honestly for them yet, and shipping a knowingly-wrong number for `INSERT` would
  violate a success criterion above. Deferred with a stated reason.
- **`SUBSCRIBE` and other streaming statements.** Excluded by an explicit rule
  rather than by accident; see "Streaming".
- **A phase breakdown.** Splitting server time into queueing, optimization and
  execution is deferred. `mz_statement_lifecycle_history` already models those
  phases (`src/adapter/src/statement_logging.rs:40-58`) and is the natural home.
- **Making MCP surface arbitrary structured data.** This design fixes MCP dropping
  notices, which is a prerequisite, but does not redesign the MCP result shape.

## Solution Proposal

Materialize measures the interval from receiving a statement to having its result
available, and reports it as an **opt-in structured notice**. Clients that ask for
it get a machine-readable payload; clients that do not see no change. The Console
consumes it and renders both numbers:

```
1.2ms server · 148ms total
```

### Where the boundary sits

**Start: per statement, not per request.** The WebSocket API accepts several
statements per request (`doc/user/content/integrations/websocket-api.md:41-45`),
and parsing happens once for the whole request
(`src/environmentd/src/http/sql.rs:1545-1585`). There is no per-statement receipt
point today, and the closest existing notion is explicitly per-request: "if there
are multiple statements in a Simple Query, then all of them have the same
`lifecycle_timestamps`" (`src/pgwire/src/protocol.rs:1095-1097`). Reusing it would
make statement 3's server time include statements 1 and 2, and exceed the client's
own total. The stamp is therefore taken **at the top of the per-statement loop body
in `execute_stmt_group`** (`src/environmentd/src/http/sql.rs:1392`).

**End: when the first result is available, before serialization.** Row-returning
statements stream one WebSocket frame per row
(`src/environmentd/src/http/sql.rs:1289-1302`), each written with an awaited
`ws.send` (`:493-499`), and `CommandComplete` is only produced once the stream
drains (`:1331-1343`). Stamping just before `CommandComplete` would fold row
serialization *and* socket-write backpressure into "server time", so a slow client
would inflate our number and, for a large result set, the two figures would
converge until the split conveyed nothing. The stamp is taken before the row loop.

The consequence is worth stating plainly: **server time does not include sending
the rows.** For a one-row `SELECT` that is immaterial. For a million-row result the
total will greatly exceed server time, and correctly so, because the remaining time
is transfer.

**Writes are excluded from this version.** For an implicit transaction, which is
what a single statement is (`src/adapter/src/session.rs:1173-1180`), the commit runs
*after* the result is returned (`src/environmentd/src/http/sql.rs:1596-1604`), and
`end_transaction` is a full coordinator round trip
(`src/adapter/src/client.rs:1072-1088`). A blind `INSERT` retires with its rows only
staged; see the comment at `src/adapter/src/coord/sequencer/inner.rs:2697-2701`.
Measuring to result-available would therefore report a sub-millisecond figure for a
write whose durable work had not happened. Rather than ship that, writes emit
nothing until the commit boundary can be included. See Open questions.

### Why the interval is "receipt to result available"

The statement lifecycle defines narrower intervals: `ExecutionBegan`,
`OptimizationFinished`, `StorageDependenciesFinished`, `ComputeDependenciesFinished`,
`ExecutionFinished` (`src/adapter/src/statement_logging.rs:40-58`). The narrowest,
`ExecutionBegan` to `ExecutionFinished`, yields the smallest and most flattering
number.

It is the wrong choice for two reasons. First, it excludes everything before
execution begins: parse, plan, optimize, and any queueing. Those are work
Materialize did, and a metric that omits them is not "how long Materialize took",
it is "how long one internal phase took". Second, it is not stable ground to build
a user-facing number on, because the phases themselves move: `enable_frontend_peek_sequencing`
now defaults to **true** (`src/sql/src/session/vars/definitions.rs:2326-2328`), so a
fast-path peek does most of its work in the Adapter Frontend rather than the
Coordinator main task (`src/adapter/src/client.rs:1456-1466`,
`src/adapter/src/frontend_peek.rs:76-80`). A metric pinned to internal phase
boundaries would silently change meaning as that work continues.

"Everything from when we received it to when we had an answer" is stable under
those changes and is what a user means by the question. It is labelled **server
time**, not "execution time", so the name does not overclaim precision about which
internal phase it covers.

### Why a notice, and not a new protocol message

A new `WebSocketResponse` variant reaches WebSocket clients and nothing else. A
notice reaches every SQL transport:

| transport | notice | new WS variant |
|---|---|---|
| pgwire (`psql`, drivers, dbt) | yes (`src/pgwire/src/protocol.rs:3412`) | no |
| WebSocket (Console) | yes | yes |
| HTTP SQL API | yes, via `SqlResult.notices` | no |
| MCP (coding agents) | after the fix below | no |

Given the direction toward headless and agent-driven use, a Console-only mechanism
is the wrong shape. There is also exact prior art: `emit_plan_insights_notice`
(`src/sql/src/session/vars/definitions.rs:1276-1281`) already ships a structured
JSON payload per statement through `AdapterNotice::PlanInsights` under a dedicated
SQLSTATE `MZ001` (`src/adapter/src/notice.rs:352`, `:543`), and the Console
dispatches on `notice.code` rather than rendering the text
(`console/src/platform/shell/CommandResultNotice.tsx:29-38`). The Console already
sets that option at handshake (`ShellWebsocketProvider.tsx:96-101`). This design
follows that pattern rather than inventing one.

**MCP currently discards notices.** `src/environmentd/src/http/mcp.rs:1061-1072`
destructures `SqlResult::Rows { rows, .. }` and `SqlResult::Err { error, .. }`,
dropping the `notices` field, and skips `SqlResult::Ok` entirely. Surfacing them is
a prerequisite here and a fix worth making regardless: an agent that cannot see
notices also cannot see plan insights or deprecation warnings.

### Why not the statement-logging machinery

The natural implementation is to read the lifecycle timestamps. It does not work.
`record_statement_lifecycle_event`
(`src/adapter/src/coord/statement_logging.rs:803-811`) requires a
`StatementLoggingId`, which exists only for statements sampled into the statement
log. Sampling is governed by `statement_logging_sample_rate` (default 0.1) and
`statement_logging_max_sample_rate` (`definitions.rs:1370-1416`), plus a byte-rate
token bucket (`src/adapter/src/statement_logging.rs:418-448`). A number that must
appear on every statement cannot be sourced from a sampled path. (The dyncfg
`ENABLE_STATEMENT_LIFECYCLE_LOGGING` defaults to true, so sampling is the real gate,
not the flag.)

Nor is `LifecycleTimestamps` available. Its doc comment scopes it to "the Adapter
frontend (`mz-pgwire`) part of the lifecycle" (`src/adapter/src/session.rs:1081-1082`),
and its only writers are in pgwire (`src/pgwire/src/protocol.rs:1346`, `:1717`). On
the HTTP and WebSocket paths `Portal.lifecycle_timestamps` stays `None` and
`began_at` falls back to `self.now()`
(`src/adapter/src/coord/statement_logging.rs:684-688`).

### Clock

Both stamps are `std::time::Instant`, reported as an integer number of
microseconds.

`EpochMillis` (`src/ore/src/now.rs:31`), which the adapter's `NowFn` returns, is
unsuitable twice over. It has millisecond granularity, so a fast peek renders as
`1ms` or `0ms`, and `0ms server` reads as broken rather than fast. And it is
wall-clock, so an NTP step can produce a negative duration. `Instant` is monotonic
by construction and sub-millisecond, matching the client half, which already uses
`performance.now()`.

Both stamps live inside `src/environmentd/src/http/sql.rs`: receipt at the
per-statement loop body (`:1392`), result-available before the send loop
(`:1164`). Nothing crosses an await, staging, or deferral boundary, so no `Instant`
needs to be threaded through the coordinator.

Note that `execute_request` and `add_result` are shared with the HTTP SQL API
through the `ResultSender` trait (`src/environmentd/src/http/sql.rs:867-897`), so an
implementation here reaches HTTP as well as WebSocket for free.

### Wire shape

A new `AdapterNotice` variant, following `AdapterNotice::PlanInsights`:

- Session variable: `emit_timing_notice`, `bool`, default `false`, matching the
  existing `emit_*_notice` naming (`definitions.rs:1276-1295`, `:1329-1334`).
- SQLSTATE: a new `MZ0xx` code alongside `MZ001`, so clients dispatch on the code
  rather than parsing text.
- Payload: JSON, `{"server_time_us": <integer>}`. Microseconds, integer, so no
  float formatting questions on the wire; clients format for display.
- One notice per statement, emitted immediately after the result is available and
  before the result is sent.

### Opt-in, and older servers

Gating on a session variable creates a problem the Console must handle.
`src/environmentd/src/http.rs:892-903` turns an unrecognized startup option into
`AdapterNotice::BadStartupSetting`, rendered as "startup setting {name} not set:
{reason}" (`src/adapter/src/notice.rs:453-455`) and pinned by
`test_ws_notifies_for_bad_options` (`src/environmentd/tests/server.rs:1273-1298`).
Startup notices are forwarded after `ReadyForQuery`
(`src/environmentd/src/http/sql.rs:387-400`), and the Console commits notices
received in `readyForQuery` or `initialState` directly into shell history
(`ShellWebsocketProvider.tsx:281-286`).

So a new Console against an older environment would print an error-looking line on
every connect. Every option the Console sends today happens to name a variable that
exists; this would be the first that might not. **The Console must filter
`BadStartupSetting` notices for options it knowingly sends.** That filter is
required for any future opt-in variable too, so it is worth having independently.

### Streaming

`SUBSCRIBE` does send `CommandComplete` when its stream ends
(`src/environmentd/src/http/sql.rs:1144-1159`), and the Console's
`commandInProgressStreaming` state handles it and stamps `endTimeMs`
(`webSocketFsm.ts:360-363`). So "emit before `CommandComplete`" would emit for
subscribes and report an entire subscribe's duration as server time. Streaming
statements are excluded by an explicit check, not by relying on them never
completing.

### Errors

Errors raised through `add_result` emit the notice: `CommandStarting` has already
been sent (`src/environmentd/src/http/sql.rs:995-1005`), so the Console's
`addErrorDuringCommandInProgress` stamps `endTimeMs`
(`webSocketFsm.ts:146-156`) and both numbers render.

Request-level errors sent bare from `run_ws_request`
(`src/environmentd/src/http/sql.rs:447-450`) do not, for example Extended-mode "each
query must contain exactly 1 statement" (`:1567-1573`). There the FSM is in
`commandSent`, whose `ERROR` transition does not complete the result
(`webSocketFsm.ts:282-291`). Nothing is displayed, which is correct: no statement
ran.

### Transactions

In WebSocket Simple mode the entire request is one transaction
(`src/environmentd/src/http/sql.rs:1407`), and a failure rolls back afterwards
(`:1415-1436`). Within an explicit `BEGIN`/`COMMIT`, each read statement reports its
own server time under the same definition as above. `BEGIN`, `COMMIT` and `ROLLBACK`
emit nothing, being writes by the exclusion above. This means an explicit
transaction shows per-read timings and no commit timing, which is consistent with
the write exclusion and should be revisited together with it.

### Display

```
1.2ms server · 148ms total          ⓘ
```

Against an environment that does not send the notice, the Console renders today's
line unchanged, which is not an error state and must not look like one:

```
Returned in 148ms                   ⓘ
```

The tooltip is rewritten for both cases. Today's wording accurately describes the
total and would become misleading beside a second number.

## Implementation Plan

Four changes across three CODEOWNERS scopes, all in this repository. Step 1 gates
steps 2 and 3; step 4 is independent.

### 1. Server: measure and emit — `@MaterializeInc/adapter`

- `src/sql/src/session/vars/definitions.rs`: add `emit_timing_notice`, default
  `false`, modelled on `emit_plan_insights_notice` (`:1276-1281`).
- `src/adapter/src/notice.rs`: add the `AdapterNotice` variant, its SQLSTATE, and
  its `Display`.
- `src/environmentd/src/http/sql.rs`: take the receipt `Instant` at `:1392`, take
  the result-available `Instant` before the send loop at `:1164`, and emit the
  notice when the variable is set, the statement is a read, and it is not
  streaming.

Tests: an integration test in `src/environmentd/tests/server.rs` alongside the
existing WebSocket message-shape tests (`:1300-1345`) asserting the notice appears
when opted in, carries a plausible payload, and that the frame sequence is
unchanged when the variable is unset. A `testdrive` case covering the pgwire path,
since that is the transport this design is largely for.

Estimate: small, mostly mechanical once the boundary decisions above are settled.

### 2. Console: consume and render — `@MaterializeInc/console`

- `ShellWebsocketProvider.tsx`: add `emit_timing_notice` to the handshake options
  (`:96-101`), and dispatch the new SQLSTATE in the notice path rather than
  committing it to history (`:281-286`).
- **Filter `BadStartupSetting` notices for options the Console sends**, so older
  environments stay clean. This is required, not optional.
- `machines/webSocketFsm.ts`, `store/shell.ts`: carry `serverTimeUs` on the command
  result.
- `timings.ts`: prefer the server value; fall back to today's computation when
  absent.
- `CommandResult.tsx`: render both numbers; rewrite the tooltip for both states.
- New `timings.test.ts`. There is no test for that module today, and the fallback
  logic is exactly the kind of branch that should have one.

### 3. MCP: stop discarding notices — `@MaterializeInc/adapter`

`src/environmentd/src/http/mcp.rs:1061-1072` drops `notices` from `SqlResult::Rows`
and `SqlResult::Err` and skips `SqlResult::Ok`. Surface them in the tool result so
agents receive this notice and every other one. Independently valuable and the
piece that makes this a platform capability rather than a Console feature.

### 4. Docs — `@MaterializeInc/docs`

- `doc/user/content/console/sql-shell.md`, currently 27 lines and silent on the
  metric, gains a short section explaining both numbers.
- The notice and the session variable are documented in the SQL reference.
- `doc/user/content/integrations/websocket-api.md` gains a forward-compatibility
  sentence advising clients to ignore unrecognized message types, which costs
  nothing here and makes future additions cheaper.

### Observability and cost

The server should record its own histogram of the measured interval. pgwire already
keeps latency histograms internally (`src/pgwire/src/protocol.rs:2722-2737`) that
never reach a client; this is the same quantity, and having it server-side lets us
answer "is the Console's number plausible" without a browser.

Cost is one extra notice per statement for opted-in sessions. On the WebSocket path
each message is one Text frame (`src/environmentd/src/http/sql.rs:493-499`), so a
one-row `SELECT` goes from four frames to five. Opt-in bounds the blast radius to
clients that asked.

### Rollout

The Console deploys independently of `environmentd`, so both states are live
simultaneously and for an unbounded period on self-managed. There is no flag day:
the Console must render correctly with and without the notice from day one, which
is why the fallback is a first-class requirement rather than a transition measure.

### Risks

| risk | mitigation |
|---|---|
| Server time looks implausibly small next to the total and reads as fake | The tooltip explains the split; this is the intended message, not a defect |
| Users read "server time" as an SLA or a benchmark | Documented as a measurement of one statement on one environment, not a guarantee |
| The write exclusion confuses users who see timings for reads but not writes | Open question below; may argue for shipping reads and writes together |
| A future phase breakdown contradicts this number | The breakdown belongs to `mz_statement_lifecycle_history`, which measures different intervals by design; documented as such |

## Minimal Viable Prototype

A Console branch that hardcodes a plausible server time and renders the two-number
footer, shared as a screenshot. Wording and ordering are the parts most likely to
attract revision and the cheapest to change before any server work starts.

## Alternatives

**A new `WebSocketResponse` variant.** A first-class message type rather than a
notice. Rejected on reach: it is structurally incapable of serving pgwire, HTTP, or
MCP clients, so it would deliver the metric to the Console and nowhere else,
permanently. It would also add to a public API's observable output.

**Replace the total with server time.** Best demo optic and closest to the literal
ticket title. Rejected: for a browser talking to a remote environment, round-trip
time is a real component of the user's experience, and hiding it trades one
inaccuracy for another.

**Tighten the client-side measurement instead.** Move the stamps closer to the
socket so JS work falls outside them. Cheap and confined to one scope. Rejected as
insufficient: it reduces noise but still measures a round trip, so it does not
answer the question, and it does nothing for non-Console clients.

**Query `mz_recent_activity_log` after execution.** No new mechanism. Rejected:
sampled and throttled as described above, requires `mz_monitor`, and adds a round
trip in order to display a latency number.

**`EXPLAIN ... WITH(TIMING)`.** Prior art worth naming: it already returns a
server-side wall clock (`src/repr/src/explain.rs:199-200`, rendered at
`src/expr/src/explain/text.rs:84-87`). Scoped to optimization only and requires
rewriting the query as an `EXPLAIN`, so it does not close CS-218, but it confirms
that reporting server-side timing to a client is not novel here.

## Open questions

**Should writes ship in this version?** Excluding them means the Shell shows a
timing for `SELECT` and nothing for `INSERT`, which is arguably worse than showing
both with a documented caveat. Including them honestly requires moving the stamp
past the implicit commit, which happens after the result is retired
(`src/environmentd/src/http/sql.rs:1596-1604`) and would need the notice emitted at
a different point in the flow. Needs an adapter opinion on whether that is a small
change or a restructuring.

**Should Query Insights follow the new number?** `PlanInsightsNotice.tsx:128` uses
the same `calculateCommandDuration` helper to decide when to surface Query
Insights. If the displayed metric changes basis, that threshold moves with it.
Recommendation: pin Query Insights to round-trip time, since a user who waited three
seconds cares that they waited three seconds. Should be an explicit choice rather
than an inherited side effect.

**Evidence for the framing.** The demo motivation comes from CS-218 and the
`#feedback` channel rather than from measurement. Before building, it is worth
confirming with a real environment what the split actually looks like for a typical
indexed peek, since the design's value rests on server time being small enough,
and the round trip large enough, for the distinction to matter.
