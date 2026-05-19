# Operation-History Discipline

A checker cannot consume what was not recorded. If a scenario claims
to test linearizability, exactly-once, no-lost-ack, or any consistency
property, an **operation history** with a known schema is the
prerequisite. Without it, the strongest verdict a run can produce is
"no obvious regression" — never "the claim survived the fault."

This reference defines the default history schema, how to record
operations, what disqualifies a history from being checkable, and how
to model ambiguous outcomes (timeouts, unknown commits, retries,
duplicate responses).

## When this discipline is mandatory

When the scenario falsifies any claim in:

- safety
- durability
- idempotency
- isolation
- ordering
- membership

For pure performance-SLO, liveness, and operational scenarios, history
discipline is optional — the existing workload + SLO oracle pattern is
sufficient.

## Default 11-field schema

Every recorded operation carries these fields. Fields the operation
type does not use are recorded as null, not omitted.

| Field | Meaning |
|---|---|
| `op_id` | Globally unique per run. UUID or monotonic counter. |
| `process_id` | Client / session / connection that issued the op. |
| `invoke_ts` | When the op was sent. Monotonic clock source preferred. |
| `complete_ts` | When the response arrived. `null` for timeouts. |
| `op_type` | `read` / `write` / `cas` / `append` / `lock` / `lease` / `join` / `leave` / `…` (model-dependent). |
| `key` | Object / session / shard the op addressed. |
| `input` | Value or arguments sent. |
| `output` | Value or response received. `null` if `complete_ts` is null. |
| `error` | Error code if the response was an error. Mutually exclusive with `output` for ops that return a value. |
| `timeout_marker` | `true` if the op timed out (response unknown). |
| `node_seen` | Which node / leader / route the client talked to. Required for replication and membership claims. |
| `fault_epoch` | Identifier of the fault window active during the op (`null` if no fault was active). |

The `fault_epoch` field is the most-skipped one and the most-load-
bearing for partition / partial-failure analyses: without it, the
checker cannot tell whether an anomaly happened under fault or not.

## Recording mechanisms

Three places to record from, with tradeoffs:

- **In-process logger.** The client library emits one log line per
  invoke and one per complete. Closest to the truth; cheapest;
  cannot capture ops that the library swallowed silently (retries
  the library hid).
- **External probe / sidecar tap.** A network-layer recorder
  intercepts client→server traffic. Catches the library-swallowed
  retries; misses anything that does not cross the network (in-
  process caches, batched writes).
- **Server-side audit log.** The server records every op it
  processed. Most accurate for what the server saw; misses ops the
  server never received (client crashes, partition drops).

For most consistency claims, **in-process logger + server-side audit
log** combined is the recommended pair: anomalies surface as
mismatches between the two.

## Complete vs weak history

A history is **complete** for a scenario if it has all the fields the
chosen checker requires. A history is **weak** if any of the
following is true:

- Missing `invoke_ts` or `complete_ts` for non-timeout ops.
- Missing `timeout_marker` on ops that did not return a response.
- Missing `fault_epoch` when at least one fault overlapped the
  workload.
- Missing `node_seen` for replication / leadership / membership claims.
- Recorded against wall-clock instead of monotonic time when the
  checker compares timestamps across processes on the same host.

A weak history produces `INCONCLUSIVE-oracle-too-weak` at run end —
the checker cannot distinguish PASS from FAIL given what was captured.
Fix the recorder before re-running.

## Ambiguous outcomes (Jepsen convention)

Real distributed systems return three things, not two: success,
failure, and *unknown*. The history must model unknowns first-class.

- **Timeout.** Op recorded with `invoke_ts` set, `complete_ts =
  null`, `timeout_marker = true`. The checker treats it as
  *could-have-succeeded* — the operation is part of the history but
  with no definite outcome. Treating timeouts as failure is a common
  bug that hides real anomalies.
- **Unknown commit.** Op recorded with an error that does not
  distinguish "did not commit" from "committed but ack lost". Same
  treatment as timeout: `timeout_marker = true`. The error code is
  retained in `error` for diagnostic purposes only.
- **Retry that succeeds after fail.** Two separate ops in the
  history with the same `input` and different `op_id`s. Never
  merged. If the system is exactly-once, the checker is responsible
  for detecting the duplicate; the recorder is not.
- **Duplicate response.** Same `op_id` appearing twice with
  different `output`s is a bug in the recorder, not the SUT.
  Different `op_id`s with the same `input` and overlapping
  `invoke_ts` are legitimate retries.

## Picker table: model → required fields

All 11 default-schema fields should be recorded for every serious
scenario; the rows below highlight the fields most critical to the
checker chosen for each model — drop them and the checker becomes
unsound for that model.

| Model | Required fields beyond the core minimum |
|---|---|
| register (single key) | `node_seen` (for per-replica linearizability) |
| map (multi-key) | `key`, `node_seen` |
| queue | `op_type` ∈ {enqueue, dequeue}, strict `invoke_ts` ordering |
| log | `op_type` ∈ {append, read}, `output` for read includes offset |
| lock | `op_type` ∈ {acquire, release}, `process_id`, `fault_epoch` |
| lease | `op_type` ∈ {acquire, renew, release}, `invoke_ts` precision matching lease TTL |
| session | `process_id` as session id; `output` of write returns version |
| membership-table | `op_type` ∈ {join, leave, view}, `node_seen` per view |
| counter | `op_type` ∈ {inc, read}, `input` = delta, `output` = post-value |
| ledger | `op_type` ∈ {credit, debit, balance}, `key` = account, idempotency id in `input` |

The model determines which checker is even applicable; the picker
table tells the recorder which fields the checker will demand.

## Anti-patterns

- **Recording only successes.** Hides timeouts and failures from the
  checker; checker sees a partial history and gives a PASS that does
  not mean what it looks like.
- **Dropping the timeout marker.** Same effect: timeouts disappear
  into the success/failure binary and the checker cannot reason
  about could-have-succeeded ops.
- **Per-client wall-clock.** Two clients with skewed clocks produce
  an op history the checker reads as inconsistent even when the SUT
  is correct. Use a monotonic source on each client, or use the
  server's receive timestamp.
- **Collapsing retries.** If the client library retries internally
  and emits one history entry covering the whole retry chain, the
  checker cannot tell exactly-once from at-least-once. Each network
  send is its own history entry.
- **Recording `output` for timed-out ops.** Checker reads this as
  "we know what happened" — but we don't. `output = null` and
  `timeout_marker = true` is the correct shape.

## In the plan

Every serious scenario's §7.M `Operation history` field declares which
of the 11 fields it captures, any extensions (extra columns the
scenario-specific checker needs), and the recording mechanism (in-
process / external / server-side / combined). If the scenario uses
the default 11-field schema unmodified, the field can simply say
"default schema, in-process + server-side." Anything else needs to be
written down.
