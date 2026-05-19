# Oracle Patterns

An oracle is the thing that says "this run passed or failed". Without
a real oracle, you have a load generator and nothing more. Most
"green but broken" results come from oracles that did not actually
run.

## Checker picker — model + claim category → checker

The plan template's §7.M block declares a `Model under test` and the
scenario's `Falsifies if it FAILs` row names a claim category. Use
this table to pick the checker(s) that match. Then read the pattern
sections below for how to actually run the chosen checker.

| Model            | Claim category       | Recommended checker(s)                                    |
|------------------|----------------------|-----------------------------------------------------------|
| register         | safety               | linearizability (Porcupine / Knossos)                     |
| log              | durability           | no-lost-ack + replay-equivalence                          |
| map              | isolation            | serializability (Elle) per-key + cross-key                |
| session          | safety               | session-consistency / monotonic-read                      |
| ledger           | idempotency          | exactly-once dedup checker on op id                       |
| membership-table | membership           | reconciliation-across-replicas at quiescence              |
| queue            | ordering             | prefix / order checker                                    |
| counter          | safety               | linearizability OR invariant-over-final-state per key     |
| lock / lease     | safety               | exclusion property + invariant-over-final-state           |

If the scenario's model is not in this table, pick the closest row
and write down in §7.M why the checker still applies (or pick "no
checker" and justify per the §7.M instructions).

## Patterns, with when to use each

### 1. Linearizability / serializability check on an op history
- **Tools:** Elle (general), Porcupine (Go), Knossos (Clojure).
- **When:** anything you claim is linearizable or serializable.
- **Inputs:** complete op history with submit + complete timestamps
  plus the value returned.
- **Output:** anomaly classification (Elle: G0, G1a, G1b, G1c,
  G-single, G2-item, G2) or a counter-example interleaving.
- **Failure mode:** truncated history is silently incomplete — the
  checker can't see what it doesn't have. Always cross-check op
  counts before trusting a PASS.

### 2. Property assertion
- **When:** an invariant holds at every step or at quiescence.
- **How:** assert in code during the run, or assert offline against
  recorded state.
- **Failure mode:** the assertion didn't run because the path
  wasn't taken. Cite execution evidence ("assertion fired N times,
  passed N").

### 3. Replay equivalence
- **When:** crash recovery, idempotent retries, fork operations.
- **How:** snapshot state A, perform op, snapshot B, replay from A,
  compare to B. They must match (allowing for documented
  non-determinism).
- **Failure mode:** the comparison is too lenient — e.g. ignores
  timestamps but should not.

### 4. Metric SLO threshold
- **When:** performance / fairness / availability scenarios.
- **How:** define p99 latency, error rate, throughput floor before
  the run. PASS = within threshold across the measurement window.
- **Failure mode:** measurement window too short, or threshold set
  from the same run that measured the baseline.

### 5. Statistical comparison against baseline
- **When:** "did this regress?" rather than "is it correct?".
- **How:** capture baseline run, capture candidate run, apply a
  test that controls false-positive rate (e.g. bootstrap CI on
  percentile of interest).
- **Failure mode:** comparing a single run against a single run.
  Always replicate.

### 6. Cross-implementation differential
- **When:** you have a reference implementation (or a model) to
  compare against.
- **How:** send the same ops to both, diff outputs.
- **Failure mode:** the reference has the same bug.

### 7. Session-consistency checker
- **When:** the claim is "reads honour the writes my session has issued so far" — read-your-writes / monotonic-read / read-after-write within a session.
- **How:** group ops by `process_id` (session). For each session, verify reads see the union of that session's prior committed writes plus anything the global order may have inserted between them. No cross-session guarantee required.
- **Inputs:** history with `process_id` and complete `invoke_ts` ordering per session.
- **Failure mode:** failing to mark session boundaries when a client reconnects; the checker treats the new connection as the same session and produces a false PASS.

### 8. Monotonic-read checker
- **When:** the claim is "once a session has read value V, it will never see an older value V' for the same key."
- **How:** for each (`process_id`, `key`), record the sequence of read outputs; assert the sequence is non-decreasing under the SUT's value ordering (last-write-wins timestamp, version vector, sequence id).
- **Inputs:** history with `process_id`, `key`, ordered reads.
- **Failure mode:** value ordering inferred wrong — e.g., comparing wall-clock timestamps in a clock-skewed cluster. Use the SUT's own version primitive, not the recorder's clock.

### 9. Prefix / order checker
- **When:** the claim is about queue / log order — "every consumer sees a prefix of the global order".
- **How:** the producer side records the canonical order; each consumer side records what it observed; assert each consumer's observation is a prefix of the canonical order (no reordering, no skips, no inserts).
- **Inputs:** producer history + per-consumer history with `op_id` cross-reference.
- **Failure mode:** unbounded reordering window — the SUT may legitimately reorder ops within an in-flight window; encode the window size and only assert prefix-of-order outside it.

### 10. No-lost-ack checker
- **When:** the claim is "every acknowledged write appears in the final state".
- **How:** scan history for ops with non-null `output` (succeeded), collect their `input` values, then query the SUT's final state at quiescence and assert every collected value is present.
- **Inputs:** history with `output` and `timeout_marker`; SUT final-state dump.
- **Failure mode:** treating timeouts (`timeout_marker = true`) as acknowledged. Timeouts are NOT acknowledged — they are unknown. The checker must ignore timed-out ops, not treat them as either acked or rejected.

### 11. Exactly-once / idempotency checker
- **When:** the claim is "the same idempotency key never produces two committed effects" (canonical example: financial ledger; also: deduplicated message handlers).
- **How:** scan history for ops sharing the same idempotency-key field in `input` (often distinct from `op_id`). Group by idempotency key and assert at most one committed effect per group in the final state (or, for non-final-state semantics, in any consumer's view).
- **Inputs:** history with the idempotency-key field annotated on every op + SUT final state or audit log.
- **Failure mode:** the idempotency key is the `op_id`. Then the checker is trivially true and tells you nothing. The key must be a *business* key the client controls and reuses across retries.

### 12. Invariant-over-final-state checker
- **When:** the claim is "after quiescence, the system satisfies invariant I" (ledger sum = 0, total tokens = supply, every joined member appears in the table once, no orphaned shards).
- **How:** drive the workload, drive the faults, wait for quiescence, dump the SUT's final state, evaluate the invariant.
- **Inputs:** final-state dump; the invariant as a predicate.
- **Failure mode:** "quiescence" is undefined. Define it as "no in-flight ops AND no pending background work AND replicas converged" and verify each separately before applying the invariant.

### 13. Reconciliation-across-replicas checker
- **When:** the claim is "at quiescence, every replica converges to the same state."
- **How:** after the workload + faults complete, force quiescence (drain in-flight, stop new ops, wait the SUT's convergence window). Dump each replica's full state. Diff pairwise.
- **Inputs:** per-replica final-state dumps; replica identity from `node_seen`.
- **Failure mode:** comparing replicas before convergence has actually completed. Some SUTs have eventual-consistency windows of minutes; convergence is a function of the SUT, not the test.

## Required for every oracle

- **Cite execution evidence in the report.** Echo the actual fact
  that the oracle ran — not just the verdict. ("Elle processed
  4,217 ops, 0 anomalies." "Property `monotonic_offset` fired
  on every append: 12,401 checks, 0 failures.")
- **Declare the oracle's scope.** Say what it cannot detect.
