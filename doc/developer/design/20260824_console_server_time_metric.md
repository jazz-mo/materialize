# Report time-to-first-row to SQL clients

- Associated: [CS-218](https://linear.app/materializeinc/issue/CS-218/console-response-time-metric-should-show-actual-database-time)

> **Revision note.** This is the third revision. Two earlier proposals were
> withdrawn after review; both failed for reasons worth recording, and they are
> kept in Alternatives so the next reader does not repeat them.

## The Problem

The Console SQL Shell reports one number, `Returned in 148ms`. It is dominated by
network round trip and browser work, so a query Materialize answered quickly
presents as a hundred milliseconds and change.

The metric is not wrong. `calculateCommandDuration`
(`console/src/platform/shell/timings.ts:28-50`) subtracts two client-side
`performance.now()` stamps. For the first statement of a command the start is
`commandSentTimeMs` (`console/src/platform/shell/machines/webSocketFsm.ts:224`);
for every later statement the start is the *previous* statement's `endTimeMs`
(`timings.ts:41-43`), so it is an inter-completion delta. The tooltip says as much.

The gap is that **Materialize measures how long it took and never tells the
client**. That is not Console-specific: a `psql` user, a dbt run and an MCP agent
are equally unable to separate their latency from ours.

## Success Criteria

- A client can obtain Materialize's own measure of a statement's duration.
- The number does not overstate Materialize's speed. It stays true when the server
  is slow, not only when it is fast.
- Delivery is deterministic: the value is unambiguously attributable to the
  statement it describes.
- Available on more than one transport, so this is a platform capability rather
  than a Console feature.
- No behaviour change for clients that do not opt in.
- The Console works, and looks deliberate, against environments predating this.

## Out of Scope

- **Statements that do not return rows.** Writes, DDL, `SET`, and transaction
  control emit nothing. This is not a workaround: the quantity proposed below is
  *time to first row*, which is undefined when there are no rows. Extending to
  those statements requires choosing a different quantity and is deferred.
- **A phase breakdown.** `mz_statement_lifecycle_history` already models phases
  (`src/adapter/src/statement_logging.rs:40-58`).
- **Changing when notices are flushed.** See "Why not the session notice queue".

## Solution Proposal

Report the interval Materialize already measures — entry to `SessionClient::execute`
until the first row arrives — to clients that opt in. The Console renders:

```
1.2ms to first row · 148ms total
```

### The measurement already exists

`SessionClient::execute` stamps `execute_started = Instant::now()` as its first
statement and returns it alongside the response
(`src/adapter/src/client.rs:809-830`). Both transports receive it: pgwire at
`src/pgwire/src/protocol.rs:1151` and `:1747`, the HTTP/WS path at
`src/environmentd/src/http/sql.rs:1721`.

Row streams are wrapped in `RecordFirstRowStream`
(`src/adapter/src/client.rs:2141-2158`), which already holds **both** stamps:
`execute_started` (`:2146`) and `recorded_first_row_instant` (`:2153`). Its `recv`
sets the latter and observes the interval into `time_to_first_row_seconds` on the
first `PeekResponseUnary::Rows` (`:2222-2229`).

That wrapper is constructed on every transport: three sites in
`src/environmentd/src/http/sql.rs` (`:1828`, `:1848`, `:1863`) and six in
`src/pgwire/src/protocol.rs` (`:2158`, `:2185`, `:2246`, `:2303`, `:2356`, `:2381`).

So this design adds no measurement. It exposes a number Materialize already
computes for its own metrics and currently discards per statement.

**What the interval covers.** It begins at entry to `execute` for an
already-bound portal, so parse and bind are excluded, and it ends when the first
row is available, so row serialization and network transfer are excluded. It is
"how long until Materialize had an answer", not "how long the statement took end
to end". The name `to first row` is chosen to say that rather than imply more.

### Why not the session notice queue

The obvious delivery mechanism, `session.add_notice` following
`emit_plan_insights_notice`, does not work, and the reason is structural.

On pgwire, `send_pending_notices()` is called **before** `send_execute_response`
(`src/pgwire/src/protocol.rs:1152-1153`, and again at `:1748-1757` on the extended
path). Rows are sent inside `send_execute_response`. A value that only exists after
the first row therefore misses that flush and is delivered on some later one,
attributed to a different statement.

On the WebSocket path, `ws_peek_result` appends drained notices *after* the
messages it was handed (`src/environmentd/src/http/sql.rs:1356-1371`), which
already contain `CommandComplete`, and the whole vector is sent at `:1164`.

`emit_plan_insights_notice` escapes this because it is emitted **before**
execution: its own description says it fires "before executing a SELECT statement"
(`src/sql/src/session/vars/definitions.rs:1279`), and it is added during sequencing
(`src/adapter/src/frontend_peek.rs:1239`). A post-execution measurement is a
different problem, and following that pattern would produce misattributed values.

### Delivery: emit at points the transports already order

Each transport constructs the completion message for a statement at a point where
it controls ordering. The value is emitted there directly, not enqueued on the
session.

- **pgwire**: the `command_complete!` macro inside `send_execute_response`
  (`src/pgwire/src/protocol.rs:2104-2114`, five invocations, one body). Emit a
  `NoticeResponse` immediately before `CommandComplete`. `send_execute_response`
  already receives `execute_started` (`:2100`).
- **WebSocket**: the `msgs` vector assembled for the statement
  (`src/environmentd/src/http/sql.rs:1356-1371`, sent at `:1164`). Insert before
  the `CommandComplete` entry.
- **HTTP SQL API**: `SqlResponse::add_result` (`:899-930`), where rows are
  collected into `SqlResult`. This is also the path MCP uses
  (`src/environmentd/src/http/mcp.rs:1035`).

This is three integration points, not one. That is the honest cost of a value that
only exists after execution begins, and it should not be described as free.

### MCP

MCP calls `execute_request` with `SqlResponse` (`mcp.rs:1035`), so it inherits the
HTTP emission point. Two further changes are needed:

- MCP currently discards notices: `mcp.rs:1061-1072` destructures
  `SqlResult::Rows { rows, .. }` and `SqlResult::Err { error, .. }` and skips
  `SqlResult::Ok`.
- An agent cannot opt in, because MCP composes the SQL itself
  (`mcp.rs:1258-1271`). Opt-in for MCP is therefore server-side configuration via
  the endpoint's `?options=` (`mcp.rs:522`), not a per-client choice. This is a
  real limitation and is called out rather than papered over.

### Wire shape

- Session variable `emit_timing_notice`, `bool`, default `false`, following the
  existing `emit_*_notice` naming (`definitions.rs:1276-1295`, `:1329-1334`).
- A dedicated SQLSTATE alongside `MZ001` (`src/adapter/src/notice.rs:352`), so
  clients dispatch on the code rather than parsing text.
- Payload JSON `{"time_to_first_row_us": <integer>}` in the notice's `detail`
  field rather than `message`, so a `psql` user is not shown raw JSON. The WS
  `Notice` struct carries `detail` (`src/environmentd/src/http/sql.rs:506-513`).
- Severity must be chosen deliberately: `Session::notice_filter`
  (`src/adapter/src/session.rs:558-564`) drops notices below the client's
  `client_min_messages`, so too low a severity means silent absence.

### Display

```
1.2ms to first row · 148ms total     ⓘ
```

Where no value arrives — an older environment, or a statement that returns no rows
— the Console renders today's line unchanged:

```
Returned in 148ms                    ⓘ
```

**These two cases are indistinguishable to the user**, which is a real weakness:
the same rendering means "this environment is old" and "this was an INSERT".
Today an `INSERT` does show a duration (`webSocketFsm.ts:176-197` stamps
`endTimeMs` on `CommandComplete`; `CommandResult.tsx:157` renders it
unconditionally), so this design introduces two display states that collide. See
Open questions.

## Implementation Plan

Separate PRs, one CODEOWNERS scope each, per `doc/developer/guide-changes.md`.

**PR 1 — measurement plumbing and pgwire emission** (`@MaterializeInc/adapter`)
Add `emit_timing_notice` (`src/sql/src/session/vars/definitions.rs`) and the
`AdapterNotice` variant with its SQLSTATE (`src/adapter/src/notice.rs`). Expose the
measured interval from `RecordFirstRowStream` and emit in `command_complete!`.
Tests: a `testdrive` case asserting the notice appears for a `SELECT` when opted
in, is absent otherwise, and is absent for a write.

**PR 2 — WebSocket and HTTP emission** (`@MaterializeInc/adapter`)
Emit at the `msgs` assembly point and in `SqlResponse::add_result`. Tests
alongside the existing WebSocket message-shape tests
(`src/environmentd/tests/server.rs:1301-1346`), including ordering relative to
`CommandComplete`.

**PR 3 — MCP** (`@MaterializeInc/adapter`)
Stop discarding notices in `mcp.rs:1061-1072`. Note this changes MCP output for
all clients, which is in tension with the "no behaviour change without opt-in"
criterion; scope it to the timing notice if that tension is not acceptable.

**PR 4 — Console** (`@MaterializeInc/console`)
Opt in at handshake, dispatch on the SQLSTATE, carry the value through the FSM,
prefer it in `timings.ts` with fallback, render, rewrite the tooltip, add
`timings.test.ts` (none exists). Also filter `BadStartupSetting` notices for
options the Console sends, since an older server rejects an unknown option name
(`src/environmentd/src/http.rs:892-903`, `src/adapter/src/notice.rs:453-455`);
that notice has SQLSTATE `SUCCESSFUL_COMPLETION` (`notice.rs:327`), so the filter
must match on message prefix, which is fragile and worth fixing at the source.

**PR 5 — Docs** (`@MaterializeInc/docs`)
`doc/user/content/console/sql-shell.md` (27 lines, silent on this metric), the
notice and session variable in the SQL reference, and a forward-compatibility
sentence in `doc/user/content/integrations/websocket-api.md`.

### Observability and cost

`time_to_first_row_seconds` (`src/adapter/src/client.rs:2226-2227`) already records
this interval, labelled by instance, isolation level and strategy
(`:2198-2208`). No new server metric is required; the existing histogram is the
check on whether client-reported values are plausible.

Cost is one extra message per opted-in statement that returns rows. On WebSocket
each message is one frame (`src/environmentd/src/http/sql.rs:493-499`).

### Rollout

The Console deploys independently of `environmentd`, so both states are live
simultaneously and indefinitely on self-managed. The fallback is a permanent
requirement, not a transition measure.

## Minimal Viable Prototype

Before any server work: measure the actual split on a real environment for a
typical indexed peek. The design's value rests entirely on time-to-first-row being
small and round trip being large. That premise is currently an inference from a
demo anecdote, and it is cheap to check. If the gap is uninteresting, none of the
rest is worth building.

## Alternatives

**Two withdrawn revisions of this document.** Recorded because both looked correct.

*Revision 1* proposed a new `WebSocketResponse` variant. Withdrawn: it can only
reach WebSocket clients, so it would have delivered the metric to the Console and
nowhere else, permanently.

*Revision 2* proposed a notice emitted from `src/environmentd/src/http/sql.rs`
while claiming pgwire reach. Withdrawn for two independent reasons. `src/pgwire`
does not depend on `mz-environmentd`, so pgwire could never have received it. And
the proposed end stamp, "before the row loop", is before the rows exist: for
`SendingRowsStreaming` the response wraps a receiver for a peek just dispatched,
which is precisely why `RecordFirstRowStream` and `time_to_first_row_seconds`
exist. A slow query would have reported a near-zero duration.

**Replace the total with the server number.** Best demo optic. Rejected: round
trip is a real part of the user's experience.

**Tighten the client-side measurement only.** Cheap, one scope. Rejected: still
measures a round trip, and does nothing for non-Console clients.

**Query `mz_recent_activity_log` afterwards.** Rejected: sampled and throttled
(`statement_logging_sample_rate` default 0.1, `definitions.rs:1370-1416`, plus a
token bucket at `src/adapter/src/statement_logging.rs:418-448`), requires
`mz_monitor`, and adds a round trip to display a latency number.

**`EXPLAIN ... WITH(TIMING)`.** Prior art (`src/repr/src/explain.rs:199-200`).
Scoped to optimization and requires rewriting the query, so it does not close
CS-218.

## Open questions

**Row-returning only is a visible inconsistency.** An `INSERT` shows a duration
today. Under this design it would show the old single number while a `SELECT`
shows two, and that fallback rendering is indistinguishable from an old server.
Options: a third display state that says "not measured"; extending to non-row
statements with a different quantity; or accepting the inconsistency. This should
be settled before implementation, not after.

**Should Query Insights follow the new number?** `PlanInsightsNotice.tsx:128` uses
the same helper to decide when to surface insights. Recommendation: pin it to
round-trip time, since a user who waited three seconds cares that they waited.
Needs to be an explicit choice.

**Is `execute_started` the right start?** It excludes parse and bind. For a
single-statement request those are small, but the value is then not "how long
Materialize took" in full. Reviewers should confirm the boundary is defensible, or
name a better one that is still available on all transports.
