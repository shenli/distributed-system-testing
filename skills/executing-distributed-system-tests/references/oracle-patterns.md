# Oracle Patterns

An oracle is the thing that says "this run passed or failed". Without
a real oracle, you have a load generator and nothing more. Most
"green but broken" results come from oracles that did not actually
run.

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

## Required for every oracle

- **Cite execution evidence in the report.** Echo the actual fact
  that the oracle ran — not just the verdict. ("Elle processed
  4,217 ops, 0 anomalies." "Property `monotonic_offset` fired
  on every append: 12,401 checks, 0 failures.")
- **Declare the oracle's scope.** Say what it cannot detect.
